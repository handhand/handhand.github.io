---
layout: post
title: "Suspend方法的Unit Test覆盖率的分析"
categories: blog, tutorial
author: handhand
---
在项目中有这样一个方法
```
class Repository {
    suspend fun retrieveData(): Result<Response> {
        // call api
    }
}
```

调用的方法像这样
```
class ViewModel {
    coroutineScope.launch {
        repository.retrieveData().fold(
            onSuccess = {
                // do something
            },
            onFailure = {
                // error handling
            }
        )
    }
}
```

在单元测试中，虽然mock了repository的这个suspend方法来对getData()进行测试，比如这样：
```
runTest {
    coEvery {
        repository.retrieveData()
    } coAnswers {
        Result.success(dummyResponse)
    }
    viewModel.getData()
    // assertions...
}
```

但是sonarqube依然提示在 `repository.retrieveData().fold( ` 这一行有两个condition，而我们只覆盖了一种condition。

通过反编译代码可以看到，因为retrieveData()是一个suspend方法，编译成bytecode之后它可能返回两种值，一种就是正常的值，这时调用者的逻辑会正常往下走；另一种情况，返回的是coroutine_suspended，这就是被suspend了，这种情况下字节码会判断，如果返回的是coroutine_suspend，会推出当前方法getData()，其实这就是挂起机制的一部分。

所有内建的suspend方法都会返回coroutine_suspended，用来标记挂起。

简单来说，在正常运行时，假设retrieveData()是一个调用IO操作的方法，那么当运行到IO操作，需要切换到IO线程时，一般会使用withContext(Dispatchers.IO)，withContext() 就是一个内建的suspend方法，它会返回coroutine_suspended，retrieveData()也会对应返回coroutine_suspended，这时调用的方法会先返回，而相关的IO操作的代码会在IO线程执行，完成后通过continuation机制重新调用我们的方法，这时retrieveData()就可以直接返回正常的值，从而继续接下来的逻辑。

如果使用上边的mock方法，只会覆盖到retrieveData()返回正常值的情况，不会覆盖挂起的情况。如果想要执行挂起的逻辑，只需要在mock里加入对内建suspend方法的调用，这样程序执行就可以走完整个挂起->continuation的流程；
```
coEvery {
    repository.retrieveData()
}.coAnswers {
    delay(1)
    Result.success(response)
}
```
