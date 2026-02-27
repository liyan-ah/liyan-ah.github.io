
  观测服务的请求调用需求是客观存在的。一般是需要观测服务的主动发起的调用信息，但是偶尔也会遇到需要观测服务被调用信息的需求。但是一般待采集的服务都是挂载在`LVS`下面的。这就势必涉及到`LVS`预设的工作模式下，一般都是`FULLNET`，需要的`real client ip`的信息获取方式。  
  笔者通过调研，实现了一种通过`BPF`来观测挂载在`LVS`下的`RS`被调用`TCP`连接信息的方式。本文中关于`toa`的操作及代码定义均引用自[Huawei/TCP_option_address](https://github.com/Huawei/TCP_option_address/tree/master)。

<!--more-->  

# 一、效果

先看下采集效果：

![upload successful](tcp-accept.png)


# 二、LVS FullNat
关于`LVS`的`Nat`,`DR`,`Tun`以及`FullNat`模式的介绍已经有了很多的资料，比如[这篇文章](https://fafucoder.github.io/2021/12/19/linux-lvs/)就介绍的很详细。这里笔者附上`FullNat`模式下的示意图：

![upload successful](full-nat.png)

如图所示，如果需要在`RS`上获取`CIP`，就涉及到`TOA`信息的解析。`TOA (tcp optional address)`是利用`tcp`协议`option`字段来传递信息的一种工作方式。关于`TOA`的约定笔者并没有找到官方的`RFC`文档。只有一些结构的定义。
```
/* MUST be 4 bytes alignment */
struct toa_data {
	__u8 opcode;
	__u8 opsize;
	__u16 port;
	__u32 ip;
};
```

同时，[rfc793](https://www.rfc-editor.org/rfc/rfc793.txt)里对`TCP header`的约定如下，理论上`toa_data`应该写在`Options`字段中。
```
    0                   1                   2                   3   
    0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |          Source Port          |       Destination Port        |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |                        Sequence Number                        |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |                    Acknowledgment Number                      |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |  Data |           |U|A|P|R|S|F|                               |
   | Offset| Reserved  |R|C|S|S|Y|I|            Window             |
   |       |           |G|K|H|T|N|N|                               |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |           Checksum            |         Urgent Pointer        |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |                    Options                    |    Padding    |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |                             data                              |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

                            TCP Header Format

```
一般来说，将`real-client ip`写入`tcp option`字段的操作是在`LVS`上进行的。而解析并且方便`RS`操作，主要是需要在`getname`的时候需要返回`real-client ip`以便于做进一步的业务逻辑，比如按照`IP`限流等，是`RS`的`toa`模块在操作的。一般是在`tcp`握手的第三个`SYN`报文处理时，`toa.ko`通过`tcp_v4_syn_recv_sock`处理的`hook`函数方式来触发`toa`数据的处理。  
这里附一段这里的逻辑：
```
static struct sock *
tcp_v4_syn_recv_sock_toa(struct sock *sk, struct sk_buff *skb,
			struct request_sock *req, struct dst_entry *dst)
#endif
{
	struct sock *newsock = NULL;

	TOA_DBG("tcp_v4_syn_recv_sock_toa called
");

	/* call orginal one */
#if LINUX_VERSION_CODE >= KERNEL_VERSION(4,4,0)
	newsock = tcp_v4_syn_recv_sock(sk, skb, req, dst, req_unhash, own_req);
#else
	newsock = tcp_v4_syn_recv_sock(sk, skb, req, dst);
#endif

	/* set our value if need */
	if (NULL != newsock && NULL == newsock->sk_user_data) {
		newsock->sk_user_data = get_toa_data(skb);
		if (NULL != newsock->sk_user_data)
			TOA_INC_STATS(ext_stats, SYN_RECV_SOCK_TOA_CNT);
		else
			TOA_INC_STATS(ext_stats, SYN_RECV_SOCK_NO_TOA_CNT);
		TOA_DBG("tcp_v4_syn_recv_sock_toa: set "
			"sk->sk_user_data to %p
",
			newsock->sk_user_data);
	}
	return newsock;
}

static void *get_toa_data(struct sk_buff *skb)
{
	struct tcphdr *th;
	int length;
	unsigned char *ptr;

	struct toa_data tdata;

	void *ret_ptr = NULL;
	unsigned char buff[(15 * 4) - sizeof(struct tcphdr)];

	TOA_DBG("get_toa_data called
");

	if (NULL != skb) {
		th = tcp_hdr(skb);
		length = (th->doff * 4) - sizeof(struct tcphdr);
		ptr = skb_header_pointer(skb, sizeof(struct tcphdr),
					length, buff);
		if (!ptr)
			return NULL;

		while (length > 0) {
			int opcode = *ptr++;
			int opsize;
			switch (opcode) {
			case TCPOPT_EOL:
				return NULL;
			case TCPOPT_NOP:	/* Ref: RFC 793 section 3.1 */
				length--;
				continue;
			default:
				opsize = *ptr++;
				if (opsize < 2)	/* "silly options" */
					return NULL;
				if (opsize > length)
					/* don't parse partial options */
					return NULL;
				if (TCPOPT_TOA == opcode &&  // 254
				    TCPOLEN_TOA == opsize) {  // 8
					memcpy(&tdata, ptr - 2, sizeof(tdata));
					TOA_DBG("find toa data: ip = "
						"%u.%u.%u.%u, port = %u
",
						NIPQUAD(tdata.ip),
						ntohs(tdata.port));
					memcpy(&ret_ptr, &tdata,
						sizeof(ret_ptr));
					TOA_DBG("coded toa data: %p
",
						ret_ptr);
					return ret_ptr;
				}
				ptr += opsize - 2;
				length -= opsize;
			}
		}
	}
	return NULL;
}

```
可以看到，这里首先调用了原有的`tcp_v4_syn_recv_sock`函数，并且在`sk_user_data`未被占用的情况下，通过`get_toa_data`的方式，从原始的`skb`中将`toa`信息解析出来，并将数据赋值给`sk->sk_user_data`。  
虽然这部分逻辑并不完全理解，但是从逻辑来看，只要读取`sk_user_data`并且判断其中是否有符合条件的值，即可获取`real-client ip`。  
至此，基本的逻辑就梳理出来了。对应的`BPF`处理逻辑也就很清晰了。

# 三、BPF 逻辑
直接上代码：

```
#define INADDR_LOOPBACK      0x7f000001 /* 127.0.0.1   */
#define INADDR_LOOPBACK_HOST INADDR_LOOPBACK
#define INADDR_LOOPBACK_NET  0x0100007f /* 127.0.0.1   */

#define ns2sec(ns) ((ns) / (1000 * 1000 * 1000))
#ifndef memcpy
#define memcpy(dest, src, n) __builtin_memcpy((dest), (src), (n))
#endif

#define MERGE_SEC 10

typedef struct {
  u8  opcode;
  u8  opsize;
  u16 port;
  u32 ip;
} toa_data_t;

// 一般 toa 模块里只会填充一个 toa 数据
#define TCP_OPTION_LEN 1

struct tcp_event {
  u32        raddr;
  u32        laddr;
  u16        rport;
  u16        lport;
  int        err;
  u64        toa_addr;
  toa_data_t toa_data;
  u64        sec;
  u64        ns;
};
typedef struct tcp_event tcp_event_t;
const struct tcp_event*  unused_0x01 __attribute__((unused));

struct {
  __uint(type, BPF_MAP_TYPE_LRU_HASH);
  __uint(key_size, sizeof(tcp_event_t));
  __uint(value_size, sizeof(u64)); // timestamp
  __uint(max_entries, 1024);
} tcp_event_map SEC(".maps");

struct {
  __uint(type, BPF_MAP_TYPE_PERF_EVENT_ARRAY);
  __uint(max_entries, 1024);
} events SEC(".maps");

enum toa_type {
  ipopt_toa        = 254, // IP_v4 客户端 IP，目前仅考虑
};

#define _AF_INET     2 /* internetwork: UDP, TCP, etc. */
#define _IPPROTO_TCP 6

SEC("kretprobe/inet_csk_accept")
int kretprobe__inet_csk_accept(struct pt_regs* ctx) {
  u64          start_ns = bpf_ktime_get_ns();
  tcp_event_t  event    = {};
  struct sock* sk       = (struct sock*)PT_REGS_RC(ctx);
  if (sk == NULL) {
    return 0;
  }

  struct sock_common sk_common = {};
  bpf_probe_read(&sk_common, sizeof(sk_common), (const void*)(sk));
  if (sk_common.skc_family != _AF_INET) {
    return 0;
  }
  // 不处理本地回环
  if (sk_common.skc_rcv_saddr == INADDR_LOOPBACK_NET ||
      sk_common.skc_daddr == INADDR_LOOPBACK_NET) {
    return 0;
  }

  event.laddr = bpf_ntohl(sk_common.skc_rcv_saddr);
  event.raddr = bpf_ntohl(sk_common.skc_daddr);
  event.lport = sk_common.skc_num;
  event.rport = bpf_ntohs(sk_common.skc_dport);

  int        err;
  toa_data_t toa_data[TCP_OPTION_LEN] = {};
  err = BPF_CORE_READ_INTO(&toa_data, sk, sk_user_data);
  if (err) {
    return 0;
  }

  u8 i = 0;
#pragma unroll
  for (i = 0; i < TCP_OPTION_LEN; i++) {
    if (toa_data[i].opcode != ipopt_toa) {
      continue;
    }
    memcpy(&event.toa_data, &toa_data[i], sizeof(toa_data_t));
  }
  u32 raddr = event.raddr;
  if (event.toa_data.ip != 0 && event.toa_data.port != 0) {
    // 挂载在 lvs 时，DS 的 IP 会发生变更。这里也给聚合掉。
    event.raddr = 0;
  }
  // remote port 都不要
  event.toa_data.port = 0;
  event.rport         = 0;
  u64  sec            = 0;
  u64  now_ns         = bpf_ktime_get_ns();
  u64* last_ns        = (u64*)bpf_map_lookup_elem(&tcp_event_map, &event);
  if (last_ns != NULL) {
    sec = ns2sec((now_ns - *last_ns));
    if (sec <= MERGE_SEC) {
      return 0;
    }
  } else {
    sec = 99;
  }
  bpf_map_update_elem(&tcp_event_map, &event, &now_ns, BPF_ANY);
  event.sec   = sec;
  event.raddr = raddr;
  u64 end_ns  = bpf_ktime_get_ns();
  event.ns    = end_ns - start_ns;

  bpf_perf_event_output(ctx, &events, BPF_F_CURRENT_CPU, &event, sizeof(event));

  return 0;
}

```

以上，周末愉快。