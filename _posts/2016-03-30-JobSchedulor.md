---
layout: post
title: JobSchedulor的简单使用
---
在android开发过程中，有种需求是在满足某种条件的时候执行任务。  
而jobSchedulor正是google官方推荐的一种使用方式。虽然现在最低版本需要21。但是总归还是了解下的。  

JobSchedulor的使用由两部分组成:JobService和jobSchedulor。  

##创建JobService
创建一个继承JobService的Service类，这个JobService运行在主线程，需要实现onStartJob(JobParameters params)和onStop(JobParameters params)  

    public class JobSchedulerService extends JobService {
    	@Override
    	public boolean onStartJob(JobParameters params) {
        	return false;
    	}

    	@Override
    	public boolean onStopJob(JobParameters params) {
        	return false;
    	}
	}

这两个方法是用来给系统调用的。当系统调用onStartJob时，返回false表示这个任务已经结束，返回true表示这个任务正要执行。
##创建JobScheduler  

    JobScheduler jobScheduler=(JobSchduler)getSystemService(Context.JOB_SCHEDULER_SERVICE);
    JobInfo.Builder builder = new JobInfo.Builder( 1,
        new ComponentName( getPackageName(), 
            JobSchedulerService.class.getName() ) );
    builder.setPeriodic( 3000 );

builder还可以设置一定的条件来触发JobService的执行。

