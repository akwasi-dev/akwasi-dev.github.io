+++
title = 'Kotlin Multiplatform: Embedding a database on iOS'
date = 2023-10-24T03:13:14Z
draft = true
+++

Over the past few days, I encountered the task of embedding a database file into a Kotlin Multiplatform (KMP) application I was working on. Fortunately, I stumbled upon [a relevant question](https://stackoverflow.com/questions/76382380/pre-populate-database-in-kmm-on-ios-side-using-sqldelight) on StackOverflow that provided a working implementation for the Android platform, but an equivalent iOS implementation was missing. This article will delve into the iOS implementation, as the Android solution has been provided referenced StackOverflow discussion.

Before we get started, I used [Moko Resources](https://github.com/icerockdev/moko-resources) to share the external database file across both platforms. Also, the app I'm building has a Share Extension that needs to tap into the same database as the main application.  From Apple's docs: 

>  Extensions are designed to be isolated from each other, from their containing apps, and from the apps that use them. They are sandboxed like any other third-party app and have a container separate from the containing app’s container.

Since I needed the Share Extension to use the same database as the main application, I had to set up an App Group for that to work. This article won't delve into app group creation but Apple has great [documentation](https://developer.apple.com/documentation/xcode/configuring-app-groups) on how to set one up in Xcode.

## Let's dive in

The code snippet below demonstrates how to implement a shared database on iOS:
```kotlin 
actual fun createDbDriver(): SqlDriver {
	val basePath = NSFileManager  
    .defaultManager  
    // Replace "group.com.app" with your actual app group
    .containerURLForSecurityApplicationGroupIdentifier("group.com.app")  
    ?.path  
  
	return NativeSqliteDriver(  
	    YourDatabase.Schema,  // Replace YourDatabase with your actual database
	    name = "databaseName.db",  
	    onConfiguration = { config ->  
	        config.copy(  
			    extendedConfig = config.extendedConfig.copy(basePath = basePath)
			)
	    }
	)
}
```
The most important piece of code is in the `onConfiguration` callback; we need that to update the location of the `.sq` file that SQLDelight would create to Apple's designated location for shared containers. 

What's essential here is the `onConfiguration` callback. It's needed so we can inform SQDelight that the database can be found at the location given to the `basePath` property. 

But the code snippet above is a good start but does not cover how to embed an existing database file. Here's the pseudocode I came up with for solving the problem: 
1. Grab the location of the external database in the assets folder
2. Define the target iOS directory to put the asset file 
3. Check if the database has already been created in the target directory, if yes, skip to step 5
4. If the file doesn't exist in the target, create the file inside the target directory
5. Initialize `NativeSqliteDriver` with the target directory as the `basePath`

Following these instructions, let's define our source and target directories:

```kotlin 
val fileManager = NSFileManager.defaultManager  // 1
val basePath = fileManager 
	.containerURLForSecurityApplicationGroupIdentifier("group.com.app")
	?.path // 2
  
val targetDbFolderPath = "$basePath/databases"  // 3
val targetDatabasePath = "$targetDbFolderPath/${DB_NAME}.db" // 4

// Using Moko-Resources
val source = MR.files.dbAsset.path 
```

From the above: 
1. We grabbed a reference to the FileManager singleton
2. We use the file manager to get access to the path for app group's folder
3. We use that path to define the intended directory for the shared database
4. Appends our database's filename to the above path. Ensure the `.db` extension is added
5. Uses moko-resources to retrieve the path of the external database we want to embed

Next, we implement steps 3 to 5 from our pseudocode:

```Kotlin 

// just for readability sake
val databaseExists = fileManager.fileExistsAtPath(targetDatabasePath) 

if (!databaseExists) { // if database file DOES NOT exist
	val folderExists = fileManager.fileExistsAtPath(targetDbFolderPath)

	if (!folderExists) {  // if database directory does not exist, create it
		fileManager.createDirectoryAtPath( // 3
			path = targetDbFolderPath, 
			withIntermediateDirectories = true, 
			attributes = null, 
			error = null
		)
	} // end of inner if-statement

	// copies the database file we want to embed from assets to the 
	// target directory 
	fileManager.copyItemAtPath(
		srcPath = source, 
		toPath = targetDatabasePath, 
		error = null
	)
} // outside if-statement

return NativeSqliteDriver(  
	YourDatabase.Schema,  // Replace YourDatabase with your actual database
		name = "databaseName.db",  
	    onConfiguration = { config ->  
	        config.copy(  
			    extendedConfig = config
					    .extendedConfig
					    .copy(basePath = targetDbFolderPath) // 5
		)
	}
)
```

I trust the above snippet is clear; however, if there's any confusion, refer back to the outlined pseudocode or reach out to me. Notably, the functions `fileManager.createDirectoryAtPath()` and `fileManager.copyItemAtPath()` return a Boolean value indicating the success or failure of the operation. Additionally, in the `onConfiguration` callback, the `basePath` property of the `extendedConfig` parameter is set to `targetDbFolderPath`. Without this adjustment, SQLDelight would default to a different operational path.


Lastly, for SQLDelight integration, ensure the external database tables are registered. Here's how:

```SQL 
CREATE TABLE IF NOT EXISTS users(  // 1
    id INTEGER PRIMARY KEY NOT NULL,  
    firstName TEXT NOT NULL,  
    lastName TEXT NOT NULL,  
    bio TEXT NOT NULL
);

 -- Write your queries here
getUsers: 
SELECT * FROM users LIMIT 10;  // 2
```

In the SQL snippet above: 
1.  We introduce the table if absent. The `IF NOT EXISTS` is vital since the table's initial creation happened during the copy process. Ensure the table and column names align with your external database.
2. With the table now recognized by SQLDelight, we can know write queries to interact with its columns. 
With the table now recognized by SQLDelight, you can formulate queries for interaction with its columns.

That wraps up this brief guide. Reach out to me if you have any questions or feedback. Thanks for reading! 