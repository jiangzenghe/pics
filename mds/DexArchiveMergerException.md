---
title: DexArchiveMergerException
date: 2018-10-28 11:43:28
tags: Android
---

### 问题描述
打包过程爆出DexArchiveMergerException。

option模块依赖option-common模块
独立的App依赖option模块

option-common模块的引用：
注意//注释的几行，这个地方可能是引起问题的关键

<!--more-->

```
dependencies {
    api 'com.google.zxing:core:3.2.1'
    api 'com.android.support.constraint:constraint-layout:1.1.3'
//    api 'com.android.support:appcompat-v7:27.0.2'
//    api 'com.android.support:design:27.0.2'
//    api 'com.android.support:cardview-v7:27.0.2'
    api 'io.reactivex:rxjava:1.1.9'
    api 'io.reactivex:rxandroid:1.2.1'
    api 'io.reactivex:rxandroid:1.2.1'
    api 'com.tbruyelle.rxpermissions2:rxpermissions:0.9.4@aar'
    api 'com.squareup.okhttp3:okhttp:3.9.1'
    api 'com.squareup.okio:okio:1.13.0'
    api 'com.squareup.okhttp3:logging-interceptor:3.9.1'
    api 'com.squareup.retrofit2:retrofit:2.1.0'
    api 'com.squareup.retrofit2:converter-gson:2.1.0'
    api 'com.squareup.retrofit2:adapter-rxjava:2.1.0'
    api 'com.squareup.retrofit2:retrofit-converters:2.1.0'
    api 'com.google.code.gson:gson:2.8.2'
    api 'com.zhy:base-rvadapter:3.0.3'
    api 'com.blankj:utilcode:1.13.6'
}
```

opion模块的引用：
```
dependencies {
    implementation 'com.android.support:appcompat-v7:27.0.2'
    implementation 'com.android.support:cardview-v7:27.0.2'
    implementation 'com.android.volley:volley:1.1.0'
    api('com.fota.android:option-common:0.0.10-SNAPSHOT@aar') { transitive = true; ; changing = true }
    api('com.fota.android:mpchart:0.0.10-SNAPSHOT@aar') { transitive = true; ; changing = true }
}
```

**细节错误信息：**

    Program type already present: android.support.v4.accessibilityservice.AccessibilityServiceInfoCompat
    Message{kind=ERROR, text=Program type already present: 
    android.support.v4.accessibilityservice.AccessibilityServiceInfoCompat, 
    sources=[Unknown source file], tool name=Optional.of(D8)}

option和option-common都是放在maven仓库的，option-common在3月12日做了提交到maven之后，3月15日把common的那几行注释了。一直没有上传。
同时在option的build中添加

    implementation 'com.android.support:appcompat-v7:27.0.2'
    implementation 'com.android.support:cardview-v7:27.0.2'
    
然后某一天，心血来潮把option-common上传了，同时option的gradle文件中加上（可能是本地依赖，没上传到maven）
                            
    implementation 'com.android.support:design:27.0.2'
依赖option仓库的app工程就出现如上下报错。


再把option打包上传到maven仓库，依赖option仓库的app构建还是报错。

(详情查看git的3.15和3.28的weOption-SDK的提交记录)

**错误：**
<font color=red>
Program type already present: android.support.design.widget.CoordinatorLayout$Behavior
Message{kind=ERROR, text=Program type already present: 
android.support.design.widget.CoordinatorLayout$Behavior, 
sources=[Unknown source file], tool name=Optional.of(D8)}
</font>

把design改成27.1.1版本,好像冲突的问题解决了。

### 原因分析：
应该还是包冲突的问题
在option和option-common所在的共同工程里，可以看到引用的包，recycleview-v7-23.4.0等
<font color=red>
注释依赖不会引起包重复报错，上面的操作不管是注释还是option和option-common的包位置对调，都不会引起包重复。
包对调指的是（appcompat-v7:27.0.2和cardview-v7:27.0.2从common移到option或者反向移动）</font>
<font color=red>
关键点在于3.28的某次更改，添加了能引起包重复的包才出现问题，不要被其他的表象混淆视听，抓住错误的关键点，才能清晰的分析错误的根本原因。
关键引起错误的包可能就是（design:27.0.2），另外注意版本的对应关系，很容易引起包冲突。</font>
<font color=red>
也有可能是在上传maven过程中传了错误版本的包，这个也会很大程度的影响逻辑的判断，鉴于当时的情况不可清楚的还原场景，只看最终结论即可。</font>

build报错信息：（记录下以方便之后再出现同样问题的核对）
```
org.gradle.api.tasks.TaskExecutionException: Execution failed for task ':app:transformDexArchiveWithExternalLibsDexMergerForDebug'.
at org.gradle.api.internal.tasks.execution.ExecuteActionsTaskExecuter.executeActions(ExecuteActionsTaskExecuter.java:100)
at org.gradle.api.internal.tasks.execution.ExecuteActionsTaskExecuter.execute(ExecuteActionsTaskExecuter.java:70)
at org.gradle.api.internal.tasks.execution.OutputDirectoryCreatingTaskExecuter.execute(OutputDirectoryCreatingTaskExecuter.java:51)
at org.gradle.api.internal.tasks.execution.SkipUpToDateTaskExecuter.execute(SkipUpToDateTaskExecuter.java:62)
at org.gradle.api.internal.tasks.execution.ResolveTaskOutputCachingStateExecuter.execute(ResolveTaskOutputCachingStateExecuter.java:54)
at org.gradle.api.internal.tasks.execution.ValidatingTaskExecuter.execute(ValidatingTaskExecuter.java:60)
at org.gradle.api.internal.tasks.execution.SkipEmptySourceFilesTaskExecuter.execute(SkipEmptySourceFilesTaskExecuter.java:97)
at org.gradle.api.internal.tasks.execution.CleanupStaleOutputsExecuter.execute(CleanupStaleOutputsExecuter.java:87)
at org.gradle.api.internal.tasks.execution.ResolveTaskArtifactStateTaskExecuter.execute(ResolveTaskArtifactStateTaskExecuter.java:52)
at org.gradle.api.internal.tasks.execution.SkipTaskWithNoActionsExecuter.execute(SkipTaskWithNoActionsExecuter.java:52)
at org.gradle.api.internal.tasks.execution.SkipOnlyIfTaskExecuter.execute(SkipOnlyIfTaskExecuter.java:54)
at org.gradle.api.internal.tasks.execution.ExecuteAtMostOnceTaskExecuter.execute(ExecuteAtMostOnceTaskExecuter.java:43)
at org.gradle.api.internal.tasks.execution.CatchExceptionTaskExecuter.execute(CatchExceptionTaskExecuter.java:34)
at org.gradle.execution.taskgraph.DefaultTaskGraphExecuter$EventFiringTaskWorker$1.run(DefaultTaskGraphExecuter.java:248)
at org.gradle.internal.progress.DefaultBuildOperationExecutor$RunnableBuildOperationWorker.execute(DefaultBuildOperationExecutor.java:336)
at org.gradle.internal.progress.DefaultBuildOperationExecutor$RunnableBuildOperationWorker.execute(DefaultBuildOperationExecutor.java:328)
at org.gradle.internal.progress.DefaultBuildOperationExecutor.execute(DefaultBuildOperationExecutor.java:199)
at org.gradle.internal.progress.DefaultBuildOperationExecutor.run(DefaultBuildOperationExecutor.java:110)
at org.gradle.execution.taskgraph.DefaultTaskGraphExecuter$EventFiringTaskWorker.execute(DefaultTaskGraphExecuter.java:241)
at org.gradle.execution.taskgraph.DefaultTaskGraphExecuter$EventFiringTaskWorker.execute(DefaultTaskGraphExecuter.java:230)
at org.gradle.execution.taskgraph.DefaultTaskPlanExecutor$TaskExecutorWorker.processTask(DefaultTaskPlanExecutor.java:123)
at org.gradle.execution.taskgraph.DefaultTaskPlanExecutor$TaskExecutorWorker.access$200(DefaultTaskPlanExecutor.java:79)
at org.gradle.execution.taskgraph.DefaultTaskPlanExecutor$TaskExecutorWorker$1.execute(DefaultTaskPlanExecutor.java:104)
at org.gradle.execution.taskgraph.DefaultTaskPlanExecutor$TaskExecutorWorker$1.execute(DefaultTaskPlanExecutor.java:98)
at org.gradle.execution.taskgraph.DefaultTaskExecutionPlan.execute(DefaultTaskExecutionPlan.java:626)
at org.gradle.execution.taskgraph.DefaultTaskExecutionPlan.executeWithTask(DefaultTaskExecutionPlan.java:581)
at org.gradle.execution.taskgraph.DefaultTaskPlanExecutor$TaskExecutorWorker.run(DefaultTaskPlanExecutor.java:98)
at org.gradle.execution.taskgraph.DefaultTaskPlanExecutor.process(DefaultTaskPlanExecutor.java:59)
at org.gradle.execution.taskgraph.DefaultTaskGraphExecuter.execute(DefaultTaskGraphExecuter.java:128)
at org.gradle.execution.SelectedTaskExecutionAction.execute(SelectedTaskExecutionAction.java:37)
at org.gradle.execution.DefaultBuildExecuter.execute(DefaultBuildExecuter.java:37)
at org.gradle.execution.DefaultBuildExecuter.access$000(DefaultBuildExecuter.java:23)
at org.gradle.execution.DefaultBuildExecuter$1.proceed(DefaultBuildExecuter.java:43)
at org.gradle.execution.DryRunBuildExecutionAction.execute(DryRunBuildExecutionAction.java:46)
at org.gradle.execution.DefaultBuildExecuter.execute(DefaultBuildExecuter.java:37)
at org.gradle.execution.DefaultBuildExecuter.execute(DefaultBuildExecuter.java:30)
at org.gradle.initialization.DefaultGradleLauncher$ExecuteTasks.run(DefaultGradleLauncher.java:314)
at org.gradle.internal.progress.DefaultBuildOperationExecutor$RunnableBuildOperationWorker.execute(DefaultBuildOperationExecutor.java:336)
at org.gradle.internal.progress.DefaultBuildOperationExecutor$RunnableBuildOperationWorker.execute(DefaultBuildOperationExecutor.java:328)
at org.gradle.internal.progress.DefaultBuildOperationExecutor.execute(DefaultBuildOperationExecutor.java:199)
at org.gradle.internal.progress.DefaultBuildOperationExecutor.run(DefaultBuildOperationExecutor.java:110)
at org.gradle.initialization.DefaultGradleLauncher.runTasks(DefaultGradleLauncher.java:204)
at org.gradle.initialization.DefaultGradleLauncher.doBuildStages(DefaultGradleLauncher.java:134)
at org.gradle.initialization.DefaultGradleLauncher.executeTasks(DefaultGradleLauncher.java:109)
at org.gradle.internal.invocation.GradleBuildController$1.call(GradleBuildController.java:78)
at org.gradle.internal.invocation.GradleBuildController$1.call(GradleBuildController.java:75)
at org.gradle.internal.work.DefaultWorkerLeaseService.withLocks(DefaultWorkerLeaseService.java:152)
at org.gradle.internal.invocation.GradleBuildController.doBuild(GradleBuildController.java:100)
at org.gradle.internal.invocation.GradleBuildController.run(GradleBuildController.java:75)
at org.gradle.tooling.internal.provider.runner.ClientProvidedBuildActionRunner.run(ClientProvidedBuildActionRunner.java:62)
at org.gradle.launcher.exec.ChainingBuildActionRunner.run(ChainingBuildActionRunner.java:35)
at org.gradle.launcher.exec.ChainingBuildActionRunner.run(ChainingBuildActionRunner.java:35)
at org.gradle.tooling.internal.provider.ValidatingBuildActionRunner.run(ValidatingBuildActionRunner.java:32)
at org.gradle.launcher.exec.RunAsBuildOperationBuildActionRunner$1.run(RunAsBuildOperationBuildActionRunner.java:43)
at org.gradle.internal.progress.DefaultBuildOperationExecutor$RunnableBuildOperationWorker.execute(DefaultBuildOperationExecutor.java:336)
at org.gradle.internal.progress.DefaultBuildOperationExecutor$RunnableBuildOperationWorker.execute(DefaultBuildOperationExecutor.java:328)
at org.gradle.internal.progress.DefaultBuildOperationExecutor.execute(DefaultBuildOperationExecutor.java:199)
at org.gradle.internal.progress.DefaultBuildOperationExecutor.run(DefaultBuildOperationExecutor.java:110)
at org.gradle.launcher.exec.RunAsBuildOperationBuildActionRunner.run(RunAsBuildOperationBuildActionRunner.java:40)
at org.gradle.tooling.internal.provider.SubscribableBuildActionRunner.run(SubscribableBuildActionRunner.java:51)
at org.gradle.launcher.exec.InProcessBuildActionExecuter.execute(InProcessBuildActionExecuter.java:47)
at org.gradle.launcher.exec.InProcessBuildActionExecuter.execute(InProcessBuildActionExecuter.java:30)
at org.gradle.launcher.exec.BuildTreeScopeBuildActionExecuter.execute(BuildTreeScopeBuildActionExecuter.java:39)
at org.gradle.launcher.exec.BuildTreeScopeBuildActionExecuter.execute(BuildTreeScopeBuildActionExecuter.java:25)
at org.gradle.tooling.internal.provider.ContinuousBuildActionExecuter.execute(ContinuousBuildActionExecuter.java:80)
at org.gradle.tooling.internal.provider.ContinuousBuildActionExecuter.execute(ContinuousBuildActionExecuter.java:53)
at org.gradle.tooling.internal.provider.ServicesSetupBuildActionExecuter.execute(ServicesSetupBuildActionExecuter.java:57)
at org.gradle.tooling.internal.provider.ServicesSetupBuildActionExecuter.execute(ServicesSetupBuildActionExecuter.java:32)
at org.gradle.tooling.internal.provider.GradleThreadBuildActionExecuter.execute(GradleThreadBuildActionExecuter.java:36)
at org.gradle.tooling.internal.provider.GradleThreadBuildActionExecuter.execute(GradleThreadBuildActionExecuter.java:25)
at org.gradle.tooling.internal.provider.ParallelismConfigurationBuildActionExecuter.execute(ParallelismConfigurationBuildActionExecuter.java:43)
at org.gradle.tooling.internal.provider.ParallelismConfigurationBuildActionExecuter.execute(ParallelismConfigurationBuildActionExecuter.java:29)
at org.gradle.tooling.internal.provider.StartParamsValidatingActionExecuter.execute(StartParamsValidatingActionExecuter.java:69)
at org.gradle.tooling.internal.provider.StartParamsValidatingActionExecuter.execute(StartParamsValidatingActionExecuter.java:30)
at org.gradle.tooling.internal.provider.SessionFailureReportingActionExecuter.execute(SessionFailureReportingActionExecuter.java:59)
at org.gradle.tooling.internal.provider.SessionFailureReportingActionExecuter.execute(SessionFailureReportingActionExecuter.java:44)
at org.gradle.tooling.internal.provider.SetupLoggingActionExecuter.execute(SetupLoggingActionExecuter.java:45)
at org.gradle.tooling.internal.provider.SetupLoggingActionExecuter.execute(SetupLoggingActionExecuter.java:30)
at org.gradle.launcher.daemon.server.exec.ExecuteBuild.doBuild(ExecuteBuild.java:67)
at org.gradle.launcher.daemon.server.exec.BuildCommandOnly.execute(BuildCommandOnly.java:36)
at org.gradle.launcher.daemon.server.api.DaemonCommandExecution.proceed(DaemonCommandExecution.java:122)
at org.gradle.launcher.daemon.server.exec.WatchForDisconnection.execute(WatchForDisconnection.java:37)
at org.gradle.launcher.daemon.server.api.DaemonCommandExecution.proceed(DaemonCommandExecution.java:122)
at org.gradle.launcher.daemon.server.exec.ResetDeprecationLogger.execute(ResetDeprecationLogger.java:26)
at org.gradle.launcher.daemon.server.api.DaemonCommandExecution.proceed(DaemonCommandExecution.java:122)
at org.gradle.launcher.daemon.server.exec.RequestStopIfSingleUsedDaemon.execute(RequestStopIfSingleUsedDaemon.java:34)
at org.gradle.launcher.daemon.server.api.DaemonCommandExecution.proceed(DaemonCommandExecution.java:122)
at org.gradle.launcher.daemon.server.exec.ForwardClientInput$2.call(ForwardClientInput.java:74)
at org.gradle.launcher.daemon.server.exec.ForwardClientInput$2.call(ForwardClientInput.java:72)
at org.gradle.util.Swapper.swap(Swapper.java:38)
at org.gradle.launcher.daemon.server.exec.ForwardClientInput.execute(ForwardClientInput.java:72)
at org.gradle.launcher.daemon.server.api.DaemonCommandExecution.proceed(DaemonCommandExecution.java:122)
at org.gradle.launcher.daemon.server.exec.LogAndCheckHealth.execute(LogAndCheckHealth.java:55)
at org.gradle.launcher.daemon.server.api.DaemonCommandExecution.proceed(DaemonCommandExecution.java:122)
at org.gradle.launcher.daemon.server.exec.LogToClient.doBuild(LogToClient.java:62)
at org.gradle.launcher.daemon.server.exec.BuildCommandOnly.execute(BuildCommandOnly.java:36)
at org.gradle.launcher.daemon.server.api.DaemonCommandExecution.proceed(DaemonCommandExecution.java:122)
at org.gradle.launcher.daemon.server.exec.EstablishBuildEnvironment.doBuild(EstablishBuildEnvironment.java:82)
at org.gradle.launcher.daemon.server.exec.BuildCommandOnly.execute(BuildCommandOnly.java:36)
at org.gradle.launcher.daemon.server.api.DaemonCommandExecution.proceed(DaemonCommandExecution.java:122)
at org.gradle.launcher.daemon.server.exec.StartBuildOrRespondWithBusy$1.run(StartBuildOrRespondWithBusy.java:50)
at org.gradle.launcher.daemon.server.DaemonStateCoordinator$1.run(DaemonStateCoordinator.java:295)
at org.gradle.internal.concurrent.ExecutorPolicy$CatchAndRecordFailures.onExecute(ExecutorPolicy.java:63)
at org.gradle.internal.concurrent.ManagedExecutorImpl$1.run(ManagedExecutorImpl.java:46)
at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1142)
at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:617)
at org.gradle.internal.concurrent.ThreadFactoryImpl$ManagedThreadRunnable.run(ThreadFactoryImpl.java:55)
at java.lang.Thread.run(Thread.java:745)
Caused by: java.lang.RuntimeException: com.android.builder.dexing.DexArchiveMergerException: Error while merging dex archives: /Users/jiangzenghe/Desktop/MyWorkspace/weoption-android/app/build/intermediates/transforms/dexBuilder/debug/0.jar, /Users/jiangzenghe/Desktop/MyWorkspace/weoption-android/app/build/intermediates/transforms/dexBuilder/debug/1.jar, /Users/jiangzenghe/Desktop/MyWorkspace/weoption-android/app/build/intermediates/transforms/dexBuilder/debug/2.jar, /Users/jiangzenghe/Desktop/MyWorkspace/weoption-android/app/build/intermediates/transforms/dexBuilder/debug/3.jar, /Users/jiangzenghe/Desktop/MyWorkspace/weoption-android/app/build/intermediates/transforms/dexBuilder/debug/4.jar, /Users/jiangzenghe/Desktop/MyWorkspace/weoption-android/app/build/intermediates/transforms/dexBuilder/debug/5.jar, /Users/jiangzenghe/Desktop/MyWorkspace/weoption-android/app/build/intermediates/transforms/dexBuilder/debug/6.jar, /Users/jiangzenghe/Desktop/MyWorkspace/weoption-android/app/build/intermediates/transforms/dexBuilder/debug/7.jar, /Users/jiangzenghe/Desktop/MyWorkspace/weoption-android/app/build/intermediates/transforms/dexBuilder/debug/8.jar, /Users/jiangzenghe/Desktop/MyWorkspace/weoption-android/app/build/intermediates/transforms/dexBuilder/debug/9.jar, /Users/jiangzenghe/Desktop/MyWorkspace/weoption-android/app/build/intermediates/transforms/dexBuilder/debug/10.jar, /Users/jiangzenghe/Desktop/MyWorkspace/weoption-android/app/build/intermediates/transforms/dexBuilder/debug/11.jar, /Users/jiangzenghe/Desktop/MyWorkspace/weoption-android/app/build/intermediates/transforms/dexBuilder/debug/12.jar, /Users/jiangzenghe/Desktop/MyWorkspace/weoption-android/app/build/intermediates/transforms/dexBuilder/debug/13.jar, /Users/jiangzenghe/Desktop/MyWorkspace/weoption-android/app/build/intermediates/transforms/dexBuilder/debug/14.jar, /Users/jiangzenghe/Desktop/MyWorkspace/weoption-android/app/build/intermediates/transforms/dexBuilder/debug/15.jar, /Users/jiangzenghe/Desktop/MyWorkspace/weoption-android/app/build/intermediates/transforms/dexBuilder/debug/16.jar, /Users/jiangzenghe/Desktop/MyWorkspace/weoption-android/app/build/intermediates/transforms/dexBuilder/debug/17.jar, /Users/jiangzenghe/Desktop/MyWorkspace/weoption-android/app/build/intermediates/transforms/dexBuilder/debug/18.jar, /Users/jiangzenghe/Desktop/MyWorkspace/weoption-android/app/build/intermediates/transforms/dexBuilder/debug/19.jar, /Users/jiangzenghe/Desktop/MyWorkspace/weoption-android/app/build/intermediates/transforms/dexBuilder/debug/20.jar, /Users/jiangzenghe/Desktop/MyWorkspace/weoption-android/app/build/intermediates/transforms/dexBuilder/debug/21.jar, /Users/jiangzenghe/Desktop/MyWorkspace/weoption-android/app/build/intermediates/transforms/dexBuilder/debug/22.jar, /Users/jiangzenghe/Desktop/MyWorkspace/weoption-android/app/build/intermediates/transforms/dexBuilder/debug/23.jar, /Users/jiangzenghe/Desktop/MyWorkspace/weoption-android/app/build/intermediates/transforms/dexBuilder/debug/24.jar, /Users/jiangzenghe/Desktop/MyWorkspace/weoption-android/app/build/intermediates/transforms/dexBuilder/debug/25.jar, /Users/jiangzenghe/Desktop/MyWorkspace/weoption-android/app/build/intermediates/transforms/dexBuilder/debug/26.jar, /Users/jiangzenghe/Desktop/MyWorkspace/weoption-android/app/build/intermediates/transforms/dexBuilder/debug/27.jar, /Users/jiangzenghe/Desktop/MyWorkspace/weoption-android/app/build/intermediates/transforms/dexBuilder/debug/28.jar, /Users/jiangzenghe/Desktop/MyWorkspace/weoption-android/app/build/intermediates/transforms/dexBuilder/debug/29.jar, /Users/jiangzenghe/Desktop/MyWorkspace/weoption-android/app/build/intermediates/transforms/dexBuilder/debug/30.jar, /Users/jiangzenghe/Desktop/MyWorkspace/weoption-android/app/build/intermediates/transforms/dexBuilder/debug/31.jar, /Users/jiangzenghe/Desktop/MyWorkspace/weoption-android/app/build/intermediates/transforms/dexBuilder/debug/32.jar, /Users/jiangzenghe/Desktop/MyWorkspace/weoption-android/app/build/intermediates/transforms/dexBuilder/debug/33.jar, /Users/jiangzenghe/Desktop/MyWorkspace/weoption-android/app/build/intermediates/transforms/dexBuilder/debug/34.jar, /Users/jiangzenghe/Desktop/MyWorkspace/weoption-android/app/build/intermediates/transforms/dexBuilder/debug/35.jar, /Users/jiangzenghe/Desktop/MyWorkspace/weoption-android/app/build/intermediates/transforms/dexBuilder/debug/36.jar, /Users/jiangzenghe/Desktop/MyWorkspace/weoption-android/app/build/intermediates/transforms/dexBuilder/debug/37.jar, /Users/jiangzenghe/Desktop/MyWorkspace/weoption-android/app/build/intermediates/transforms/dexBuilder/debug/38.jar, /Users/jiangzenghe/Desktop/MyWorkspace/weoption-android/app/build/intermediates/transforms/dexBuilder/debug/39.jar, /Users/jiangzenghe/Desktop/MyWorkspace/weoption-android/app/build/intermediates/transforms/dexBuilder/debug/40.jar, /Users/jiangzenghe/Desktop/MyWorkspace/weoption-android/app/build/intermediates/transforms/dexBuilder/debug/41.jar, /Users/jiangzenghe/Desktop/MyWorkspace/weoption-android/app/build/intermediates/transforms/dexBuilder/debug/42.jar, /Users/jiangzenghe/Desktop/MyWorkspace/weoption-android/app/build/intermediates/transforms/dexBuilder/debug/43.jar, /Users/jiangzenghe/Desktop/MyWorkspace/weoption-android/app/build/intermediates/transforms/dexBuilder/debug/44.jar, /Users/jiangzenghe/Desktop/MyWorkspace/weoption-android/app/build/intermediates/transforms/dexBuilder/debug/45.jar, /Users/jiangzenghe/Desktop/MyWorkspace/weoption-android/app/build/intermediates/transforms/dexBuilder/debug/46.jar, /Users/jiangzenghe/Desktop/MyWorkspace/weoption-android/app/build/intermediates/transforms/dexBuilder/debug/47.jar, /Users/jiangzenghe/Desktop/MyWorkspace/weoption-android/app/build/intermediates/transforms/dexBuilder/debug/48.jar, /Users/jiangzenghe/Desktop/MyWorkspace/weoption-android/app/build/intermediates/transforms/dexBuilder/debug/49.jar, /Users/jiangzenghe/Desktop/MyWorkspace/weoption-android/app/build/intermediates/transforms/dexBuilder/debug/50.jar, /Users/jiangzenghe/Desktop/MyWorkspace/weoption-android/app/build/intermediates/transforms/dexBuilder/debug/51.jar, /Users/jiangzenghe/Desktop/MyWorkspace/weoption-android/app/build/intermediates/transforms/dexBuilder/debug/52.jar, /Users/jiangzenghe/Desktop/MyWorkspace/weoption-android/app/build/intermediates/transforms/dexBuilder/debug/53.jar, /Users/jiangzenghe/Desktop/MyWorkspace/weoption-android/app/build/intermediates/transforms/dexBuilder/debug/54.jar, /Users/jiangzenghe/Desktop/MyWorkspace/weoption-android/app/build/intermediates/transforms/dexBuilder/debug/55.jar, /Users/jiangzenghe/Desktop/MyWorkspace/weoption-android/app/build/intermediates/transforms/dexBuilder/debug/56.jar, /Users/jiangzenghe/Desktop/MyWorkspace/weoption-android/app/build/intermediates/transforms/dexBuilder/debug/57.jar, /Users/jiangzenghe/Desktop/MyWorkspace/weoption-android/app/build/intermediates/transforms/dexBuilder/debug/58.jar, /Users/jiangzenghe/Desktop/MyWorkspace/weoption-android/app/build/intermediates/transforms/dexBuilder/debug/59.jar
at com.android.builder.profile.Recorder$Block.handleException(Recorder.java:55)
at com.android.builder.profile.ThreadRecorder.record(ThreadRecorder.java:104)
at com.android.build.gradle.internal.pipeline.TransformTask.transform(TransformTask.java:212)
at sun.reflect.GeneratedMethodAccessor449.invoke(Unknown Source)
at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
at java.lang.reflect.Method.invoke(Method.java:498)
at org.gradle.internal.reflect.JavaMethod.invoke(JavaMethod.java:73)
at org.gradle.api.internal.project.taskfactory.IncrementalTaskAction.doExecute(IncrementalTaskAction.java:46)
at org.gradle.api.internal.project.taskfactory.StandardTaskAction.execute(StandardTaskAction.java:39)
at org.gradle.api.internal.project.taskfactory.StandardTaskAction.execute(StandardTaskAction.java:26)
at org.gradle.api.internal.tasks.execution.ExecuteActionsTaskExecuter$1.run(ExecuteActionsTaskExecuter.java:121)
at org.gradle.internal.progress.DefaultBuildOperationExecutor$RunnableBuildOperationWorker.execute(DefaultBuildOperationExecutor.java:336)
at org.gradle.internal.progress.DefaultBuildOperationExecutor$RunnableBuildOperationWorker.execute(DefaultBuildOperationExecutor.java:328)
at org.gradle.internal.progress.DefaultBuildOperationExecutor.execute(DefaultBuildOperationExecutor.java:199)
at org.gradle.internal.progress.DefaultBuildOperationExecutor.run(DefaultBuildOperationExecutor.java:110)
at org.gradle.api.internal.tasks.execution.ExecuteActionsTaskExecuter.executeAction(ExecuteActionsTaskExecuter.java:110)
at org.gradle.api.internal.tasks.execution.ExecuteActionsTaskExecuter.executeActions(ExecuteActionsTaskExecuter.java:92)
... 107 more
```

**涉及到的问题解决方面结论：**

    一个是jar包重复到问题，在AS中使用gradel命令查看依赖树./gradlew -q :app:dependencies，另外要学会exclue去除不需要的包。
    另一个问题是api依赖和implementation依赖到理解，api会传递依赖自己的父依赖到子（引用ta的）module，但是implementation只对子module（引用ta的module）暴露本身。
    最后需要对各版本的补充库、兼容库等有个大概等了解认识。