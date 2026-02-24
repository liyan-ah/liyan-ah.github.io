+++
title = "一种责任链模式的实现"
description = "责任链模式是代码实现中经常使用的设计模式。在涉及到策略处理或者其他的接口实现时，使用责任链模式可以大幅的降低处理的复杂度，而且可以让代码变得更加优美。"
date = 2026-02-12T11:40:14+08:00
license = ""
hidden = false
comments = true

[taxonomies]
categories = ["代码之美"]
tags = ["Rust", "设计模式", "责任链模式"]
+++

> 许久没有写博客了，25年全年都没有更新。估计26年上半年还会继续占用时间，想写点东西还是比较困难的。今天整理了下遇到的比较优美的代码，发出来赏析下。

责任链模式的常见使用场景是对目标数据需要进行多个独立规则的校验，如请求参数能否解析、参数中的业务指标是否非法、权限信息是否正常等。这种需求一般可以将校验的过程抽象成一个接口，然后将每个校验操作都封装成一个接口的实现，在校验管理器中维护一个注册、校验函数。但是这种方法并不优美，尤其是对于`Rust`语言来说，`Rust`本身的链式操作是很优美的，将这些校验操作进行封装，通过链式操作来生效则会比较优雅。

如下是其中的一个通用实现。这里`Handler`是抽象出来的处理接口，而`HandlerA`是其中的一个实现。通过定义`HandlerChainExt` 接口以及 `Chain` 结构体，即可实现在不改变`Handler`定义以及`HandlerA`实现的前提下完成链式操作。这无疑是非常优雅的。而且通过下面的单测示例可以看到，`.and_then`的调用方式及其简介、明了。

```Rust
#[cfg(test)]
mod tests {
    use std::fmt::Debug;

    use anyhow;
    use tracing::tracing::debug;

    #[async_trait]
    trait Handler: Debug + Send + Sync {
        type WriteInput: Debug + Send + Sync;
        type WriteOutput: Debug + Send + Sync;
        async fn check(&self, input: Self::WriteInput) -> anyhow::Result<Self::WriteOutput>;
    }

    #[derive(Debug)]
    struct HandlerA {
        output: String,
    }

    impl HandlerA {
        fn new(output: String) -> Self {
            Self { output }
        }
    }

    #[async_trait]
    impl Handler for HandlerA {
        type WriteInput = String;
        type WriteOutput = String;

        async fn check(&self, input: String) -> Result<String, anyhow::Error> {
            debug!("handler A, {input}");
            println!("handler A, input: {input}");
            println!("hanlder A, output: {input}|{}", self.output);
            Ok(format!("{input}|{}", self.output))
        }
    }

    trait HandlerChainExt: Handler + Sized {
        fn and_then<T>(self, next: T) -> Chain<Self, T>
        where
            T: Handler,
        {
            Chain {
                first: self,
                second: next,
            }
        }
    }

    impl<T> HandlerChainExt for T where T: Handler {}

    #[derive(Debug)]
    struct Chain<T, U> {
        first: T,
        second: U,
    }

    impl<T, U> Chain<T, U> {
        #[allow(dead_code)]
        fn new(first: T, second: U) -> Self {
            Self { first, second }
        }
    }

    #[async_trait]
    impl<T, U> Handler for Chain<T, U>
    where
        T: Handler,
        U: Handler<WriteInput = T::WriteOutput>,
    {
        type WriteInput = T::WriteInput;
        type WriteOutput = U::WriteOutput;

        async fn check(&self, input: Self::WriteInput) -> anyhow::Result<Self::WriteOutput> {
            let output = self.first.check(input).await?;
            self.second.check(output).await
        }
    }

    #[tokio::test]
    async fn test_handler_chain() {
        let handler = HandlerA::new("aa".into())
            .and_then(HandlerA::new("bb".into()))
            .and_then(HandlerA::new("cc".into()));
        assert!(handler
            .check("raw_input".into())
            .await
            .is_ok_and(|x| x == "raw_input|aa|bb|cc"));
    }
}
```

`Rust`语言的特点使得很多代码实现显得非常优雅，后面再找一些。