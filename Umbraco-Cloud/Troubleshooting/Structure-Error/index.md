# Troubleshooting structure deployment/restore errors

## Error in files containing site structure

On some occassions it's possible that you'll encounter collision errors on your Umbraco Cloud environments. This means that two `.uda` files are created for the same entity type. `.uda` files contains schema for each of your entity types (e.g Document Types, Templates, Macros, Dictionary Items, Datatypes).

Example:

    Some artifacts collide on unique identifiers.
    This means that they have different Udis, yet
    they refer to the same unique Umbraco object
    and therefore cannot be processed.
    ---------------------------------------------
    Collisions for entity type "document-type": 
      Collisions for unique identifier "home":
        UdaFile: ~/data/revision/document-type__4c04d968448747d791b5eae254afc7ec.uda
        UdaFile: ~/data/revision/document-type__f848c577f02b4ee5aea84f87458072a4.uda


In the example above there are two files in the `~/data/revision` folder which contain the same alias, `unique identifier`, for a Document Type. Each file has a different name, but the Document Type they refer to is both `home`. When Deploy tries to create the site structure from these files, it is inspecting each file to try to create a Document Type with a unique alias.

In the case above, there are two files who share the same alias which leads to a conflict: it's impossible for Deploy to know which is the "correct" file, so it gives up and sends an error back.

### Cause

The main cause of this problem is when a Document Type (or Media Type, Datatype, etc) is manually created in two environments using the same alias. 

If you have two or more Cloud environments, we recommend that you never create schema or make schema changes directly on the Live or Staging environments. You should work with schema only on your Development environment or even better, your local clone of the project.

### Fixing

In order to fix this problem you will have to decide which Document Type is the correct one. The error message will give you a lot of details you can use in your investigation:
  * The affected entity type (Document Type, Datatype, Member type, etc.)
  * The `unique identifier` (alias)
  * A list of the files containing the same `unique identifier`


Follow these simple steps to get your project back on track:

1. Clone down the affected environment - if all environments are affected, you only need to clone down the Development environment
    * You do not need to run the project locally. Cloning down is simply to add the file deletions to Git
2. Compare the colliding `.uda` files to determine which file contains the most correct data - Use a file comparison tool like *DiffMerge* or *WinMerge* for this
3. Delete the `.uda` file(s) you believe to be the wrong one(s)
    * If you are unsure which file is the correct one, take a backup of the `/data/revision` folder so you can always restore the files you've deleted
4. Use Git to commit and push the deletion to your Cloud environment
5. If you have more than one Cloud environment affected by the collision issue, be sure to deploy the deletion all the way up to the Live environment

### Additional notes

Sometimes you might need to run another extraction on your Cloud environment after deploying in order to get a `deploy-complete` marker in your `/data/revision` folder and turn your environment *green*. To do this, follow these steps:

1. Access **Kudu** on the affected environment
2. Use the CMD console (found under the 'Debug console' menu) to navigate to your `site/wwwroot/data/` folder
3. In the console, type the following command: `echo > deploy`
4. When the extracion is done, you should see a `deploy-complete` marker, which means the extraction error was succesful


## Error in Courier files containing site structure (Courier)

> ***NOTE:** The following troubleshooting is for Umbraco Cloud projects using Courier.*

On some occassions it's possible that you'll encounter an "Error in Courier files containing site structure". This usually means that two Courier files are created for the same entity type.  

For example: there are two files in the `~/data/Revision/documenttypes` folder which contain the same alias for a document type, let's say `home`. Each file has a different file name but the document type they refer to is both `home`. When Courier tries to create the site structure from those files it is expecting each file to try to create a document type with a unique alias.

If two files share that same alias this leads to a conflict, it's impossible for Courier to know which is the "correct" one, so it has to give up and send an error back.

### Cause

The main cause of this problem is when a document type (or media type or member type) gets manually created on two environments using the same alias. If you're new to Umbraco Cloud, for example, and have been using Umbraco for a while, it might actually be surprising that Umbraco takes care of syncing a document type between environments. You might have decided to create the same document type manually in each environment because that's what you're used to doing. 

### Fixing

In order to fix this conflict you will have to decide which document type is "the most correct" one. For some help with that the list of all properties in each Courier file will be logged in your `~/App_Data/Logs/CourierTraceLog.txt` file when you see the `Error in Courier files containing site structure` error. This might help you determine: the `home` document type with the extra 3 properties that I added today is the one that I want to have in all of my environments. You can then remove the non-relevant entity type from git and deploy your changes to the next environment. This should clear up the error and allow you to move on. The log file will list the file names for each conflicting file. 

### Troubleshooting using SQL queries

Sometimes it's hard to determine by eye which one of your document types is "correct" and you might want to have a look in your SQL database to see which one you want to keep. In that case you can perform a SQL query to find the content type and it's properties so you can compare them.

The courier files in `~/data/Revision/` are usually named after the unique Id of the thing you're trying to find, so in the case of content types you could use the file name (without the `.courier`) extension to find the corresponding type in the database. So if the filename is `efc3208b-efc6-44f8-928c-12c03ccf4700.courier` you might query the database like so:

    SELECT Alias, Name, UniqueID
      FROM cmsPropertyType
      WHERE contentTypeId IN 
      (SELECT umbracoNode.id FROM umbracoNode WHERE uniqueID = 'efc3208b-efc6-44f8-928c-12c03ccf4700')
      ORDER BY Alias

This will give you the properties available on this content type in this environment and you can compare the properties to the ones in your log file to see if the environment you're perfoming the SQL query in is correct according to what you want the content type to be.

Again, once you've decided which Courier file in `~/App_Data/Revision/` contains the best representation of your content type then you can remove the duplicate from disk and commit the removal to git. This should unblock your future deploys from popping up this error.

It's important to note that only you can decide which document type you want to keep, Courier can not guess which document type you think is the most complete.
