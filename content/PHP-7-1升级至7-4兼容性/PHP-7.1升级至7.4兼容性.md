<!--more-->
# each弃用
从`php7.2`开始，官方开始弃用`each`函数。作为一个伪码农我是很震惊的，无法想象哪天`python`把`range`弃用了，代码维护人员是否有毅力将所有的历史内容都重新适配下。`php官方`就是这么<del>任性</del>自信，直接删除，不做兼容。网上找到的兼容方案是这样的：
```  
function func_new_each(&$array){
    $res = array();
    $key = key($array);
    if($key !== null){
        next($array);
        $res[1] = $res['value'] = $array[$key];
        $res[0] = $res['key'] = $key;
    }else{
        $res = false;
    }
    return $res;
}
 
 
// 替换前
list($scalar_type, $scalar_value) = each($methName->me);
 
 
// 替换后
list($scalar_type, $scalar_value) = func_new_each($methName->me);

```

# mcrpt库弃用
这个更狠了，整个库直接弃用。改为推荐[`openssl`](https://www.php.net/manual/zh/ref.openssl.php)。好在工程不是长期维护的，后续可能还有重构的规划，所以以解决问题为优先目标吧，直接将废弃的`mcrypt`作为插件，重新编入`php7.4`:

```  
#! /bin/bash
 
# any problem please contact me
# used to install mcrypt.so extentions for php
 
php_path=${1:-"/usr/local/php"};
echo "install php under ${php_path}";
 
function Info(){
    echo `whoami`@`hostname`:`pwd`;
}
 
function check_env(){
    mised=1;
    for i in {0}; do
        set -x;
        command -v ${php_path}/bin/php >/dev/null 2>&1 || break
        command -v ${php_path}/bin/phpize >/dev/null 2>&1 || break
        command -v ${php_path}/bin/php-config >/dev/null 2>&1 || break
        set +x;
        mised=0;
    done
    return ${mised};
}
 
check_env;
if [ $? -ne 0 ]; then
    echo "missing php exe file";
    exit 233;
fi
 
# wget mcrypt
wk_dir="${HOME}/php_extend";
mkdir -p ${wk_dir};
if [ $? -ne 0 ]; then
    echo "cannot create ${wk_dir}, check permission";
    exit 20;
fi
rm -rf ${wk_dir}/*;
cd ${wk_dir} && wget http://pecl.php.net/get/mcrypt-1.0.4.tgz;
if [ $? -ne 0 ]; then
    echo "download mcrypt-1.0.4 failed";
    exit 23;
fi
 
# prepare
cd ${wk_dir} && tar -xvf mcrypt-1.0.4.tgz;
cd ${wk_dir}/mcrypt-1.0.4 && ${php_path}/bin/phpize;
if [ $? -ne 0 ]; then
    echo "mcrypt path init failed";
    exit 233;
fi
 
# configure
cd ${wk_dir}/mcrypt-1.0.4 && ./configure --prefix=${wk_dir}/mcrypt --with-php-config=${php_path}/bin/php-config;
 
if [ $? -ne 0 ]; then
    echo "=========> attention that, configure failed, would try to make";
fi
 
# XXX: there is a sudo
cd ${wk_dir}/mcrypt-1.0.4 && make && sudo make install
 
# check if has output
extend_dir=$(${php_path}/bin/php-config --extension-dir);
if [ ! -f "${extend_dir}/mcrypt.so" ]; then
    echo "======>  mcrypt.so generate failed";
    exit 233;
fi
 
ini_dir=$(${php_path}/bin/php-config --ini-path);
if [ ! -f "${ini_dir}/php.ini" ]; then
    echo "======> missing php ini file, ${ini_dir}/php.ini";
    exit 233;
fi
 
cat << EOF
mcrypt.so is generated in ${extend_dir}/mcrypt.so,
please do with sudo permission:
echo "extension=mcrypt.so" >> ${ini_dir}/php.ini
 
EOF
 
# XXX: there is a sudo
# need sudo, and should run ok
#sudo echo "extension=mcrypt.so" >> ${ini_dir}/php.ini
#if [ $? -ne 0 ]; then
#    echo "cannot write `extension=mcrypt.so` into ${ini_dir}/php.ini, please check permission";
#    exit 2333;
#fi
#
#echo "mcrypt.so install ok";  

```

语言的兼容性确实像噩梦一样。希望不要老是整这些幺蛾子。

# 参考文献
[php7.2 废弃each方法
](https://blog.csdn.net/zchare/article/details/81903362)
[php7.2 安装 mcrypt扩展
](https://blog.csdn.net/cbuy888/article/details/93618842)