# Android Application Extension Library


**I'm still working on this library (90/100), the sources will be soon on GitHub.**

##Include the library

1. Add the **AAEL.jar** to the build path of your app
2. Extends the `AAELApplication` class (For more info see [Application](http://developer.android.com/reference/android/app/Application.html) docs)

		package foo.bar;
		
		public class YourApplicationClass extends AAELApplication { ... } 

3. Declare the full qualified name of your `EchlabSWApplication` subclass implemented for the application into the AndroidManifest.xml
	
		<application
        	...
        	android:name="foo.bar.YourApplicationClass"
        	...>
        </application>

Here we go! Now you are ready to use the library.	


##ServiceManager
The library offers a `ServiceManager` that implements the following features:

* Synchronization between remote services
* IPC communication between remote services (each service can communicate with any other)
* Synchronous service's binding (no more callback)

With this manager you will never have to bind manually remote services using [ServiceConnection](http://developer.android.com/reference/android/content/ServiceConnection.html) and use the callback functions _onServiceConnected()_, _onServiceDisconnected()_ that makes you to write a lot of boilerplate code. As you should know, the service binding process is an asynchronous operation and composed by these main events:

* Service binding (asynchronous action)
* Service connection and get service reference
* Remote method call (synchronous action, only if the method is not declared _oneway_)

Using the AAEL's `ServiceManager` the process binding become a **synchronous** operation, that because when the application starts, the service manager bind all the others declared services in the Manifest (a modal progress dialog is shown on the screen in the meantime until all the services are instancied and bound). 
Each remote service can talk with each other using the ServiceManager.

The remote services creation and destruction and the whole remote services life cycle is managed by the `ServiceManager`, so everything is ready after that the `AAELApplication.onCreate()` method is called, immediately after the application is launched, in that way we are sure that all services are working and are up.


If a service is killed before a remote call, the ServiceManager create a new instance and bind the service before invoke the method, making this operation transparent to the caller. The caller don't needs to take care about the service status or service creation, it only needs to invoke the desidered remote method and if the remote service is down, the ServiceManager will create and bind the service before calling the method.


###How-To use the ServiceManger

All the services are loaded using this function:
	
	ServiceManager.loadServices(onServiceListener);
	
so you should call it before than anything else, for example in the MainActivity or during the splash screen, anyway in your first *Activity* or your first *Service* that is not an *AAEL Service*.
	
the function takes a listener as parameter that will be notified on these events:

* `onLoading`: The services loading is in progress
* `onServiceLoaded`: All the services are up and full working
* `onFailed`: error, not possible to load the services


		private OnServiceListener onServiceListener = new OnServiceListener.Stub() {
			@Override
			public void onServiceLoaded() throws RemoteException { }

			@Override
			public void onLoading() throws RemoteException { }
	
			@Override
			public void onFailed() throws RemoteException { }

		};

After the callback `onServiceLoaded` function is called, you are sure that all the remote services are up and you can continue to start your app.

*All the service's methods are called on the main-thread.*

ServiceManager is itself a service so you need to declare it in the Applicaion Manifest, 

	<service android:name=".service.ServiceManager" />

with the restriction that it cannot be declared as remote, it needs to run in the same process of the main application.
	
#####Add a new remote service 

The `ServiceManager` handle all the remote services declared in the Application Manifest with meta-data *aael_service*. 

To add a new remote service:

1. Create your service and extends the `AAELService` class and implements the `onBind()` methods returning a IBinder through which clients can call on to the service.
2. Create an AIDL file and put it in a sub-pakage `aidl` where the service is declared.

	For example: If your service is declared in the package `com.package.myservice`, so you should put its AIDL file in the package `com.package.myservice.aidl`.

3. Declare your service in the Manifest


		<service android:name="com.package.example.ServiceName">
			<meta-data android:name="aael_service" android:value="ServiceName"/>
		</service>


#####Call a remote service method

* If your service is remote (a new process is created for the service): a new thread is spawn by the framework in the remote process and the method is executed in the new thread just created. 	
For example: If in the application you create 5 thread and for each thread you invoke a remote service, so in the service will be spawn respectively 5 new thread and each method will be invoked in their respective thread.
**(so make sure that the methods implementation is thread-safe)**
* If the service is local (not remote): the method is invoked in the caller thread **(make attention to UI Main thread)**.
 
#####Foreground Service
If you need that your service is executed in [Foreground](http://developer.android.com/guide/components/services.html#Foreground), you can add the meta-data `isForeground` passing as value `true`.

	<meta-data android:name="isForeground" android:value="true|false"/>
	
Since Android 4, if the service is flagged as inForeground, an icon is shown in the notification bar, until the service is alive.



##LoggerService

Is the service that manage the Log writing on File in the directory `/sdcard/<ApplicationName>/logs`, it handle the log daily rotation too.
To use the LoggerService in your app, you should: 

	
* Declare the service:
		
		<service
            android:name="com.aael.android.common.lib.utils.log.LoggerService"
            android:process=":logger_service" >
            <meta-data
                android:name="aael_service"
                android:value="LoggerService" />
        </service>
        
Add the *WRITE\_EXTERNAL\_STORAGE* in the Manifest:

	<uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE" />
	
The default log level is INFO, you can change it calling the method:

		Logger.init(LogLevel level)
	
#####Uncaught Exception
The *LoggerService* handle Uncaught Exceptions, so if your app crashes for some reason, you will find a new file (one for each crash) in the log directory with the crash exception logged.

