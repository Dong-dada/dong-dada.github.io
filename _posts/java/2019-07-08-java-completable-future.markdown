

## Future

> A {@code Future} represents the result of an asynchronous computation.  Methods are provided to check if the computation is complete, to wait for its completion, and to retrieve the result of the computation.

- Future 代表异步计算的结果。
- Java 中提供的 Future 接口中，提供了 查询计算是否完成、等待计算完成、获取计算结果 等简单的几个方法。
- 可以像以下例子中所示，向 Executor 提交一个 Callable，通过 Future 来获取异步执行的结果:

```java
public static void main(String[] args) {

    ExecutorService executor = Executors.newSingleThreadExecutor();
    Future<String> future = executor.submit(new Callable<String>() {
        @Override
        public String call() throws Exception {
            return Thread.currentThread().getName();
        }
    });

    //在我们异步操作的同时一样可以做其他操作
    doSomethingElse();
    
    try {
        // 尝试获取异步执行的结果
        String res = future.get();
    } catch (InterruptedException e) {
        e.printStackTrace();
    } catch (ExecutionException e) {
        e.printStackTrace();
    }
}
```

## CompletableFuture

> A Future that may be explicitly completed (setting its value and status), and may be used as a CompletionStage, supporting dependent functions and actions that trigger upon its completion.

- CompletableFuture 是一种特殊的 Future。
- 它可以显示地被完成。


```java
// supplyAsync 工厂方法，可以生成一个 CompletableFuture 对象
CompletableFuture<String> completableFuture = CompletableFuture.supplyAsync(() ->{
    // 执行耗时操作...
    return result;
});

// get 方法，
String result = comletableFuture.get();
```