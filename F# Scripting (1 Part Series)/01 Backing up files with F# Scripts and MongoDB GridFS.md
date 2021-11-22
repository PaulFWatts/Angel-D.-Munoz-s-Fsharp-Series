![Cover image for Backing up files with F# Scripts and MongoDB GridFS](https://res.cloudinary.com/practicaldev/image/fetch/s--5cxHMV3s--/c_imagga_scale,f_auto,fl_progressive,h_420,q_auto,w_1000/https://dev-to-uploads.s3.amazonaws.com/i/ig1uhqq2c35ak8lysnre.png)

[![Angel D. Munoz](https://res.cloudinary.com/practicaldev/image/fetch/s--FXr-t6j_--/c_fill,f_auto,fl_progressive,h_50,q_auto,w_50/https://dev-to-uploads.s3.amazonaws.com/uploads/user/profile_image/35800/b392acdc-d3fc-4ab0-a1cc-4d200e1a4582.jpg)](https://dev.to/tunaxor)

F# 5.0 has strong scripting capabilities thanks to the updated `#r "nuget: "` directive, this allows you to pull any NuGet dependency and start using it from your F# scripts right away. To show off this feature I'll propose the following use case

> I want to backup files from a directory into a database and be able to restore them back to the directory I want

There are some ways we can do this from saving the data to a database and uploading to AWS/GCP/Azure/Dropbox in this case I wanted to showcase one of the Libraries I worked on previously called [Mondocks](https://github.com/AngelMunoz/Mondocks) which allows you to write MongoDB commands and execute them via the normal .NET MongoDB Driver.  
I have a Raspberry PI on my LAN with a 1TB HDD and it has docker running so I can save my files there.

Let's start with the Backups, also I added some comments so you can read the source code and continue reading hereafter

#!/usr/bin/env \-S dotnet fsi

#r "nuget: MongoDB.Driver"

#r "nuget: MongoDB.Driver.GridFS"

#r "nuget: Mondocks"

#r "nuget: Spectre.Console"

open System

open System.IO

open Spectre.Console

open MongoDB.Bson

open MongoDB.Driver

open MongoDB.Driver.GridFS

open Mondocks.Types

open Mondocks.Queries

let args \= (fsi.CommandLineArgs)

// here we also start by parsing the cli args

let restoreTo \=

args

|> Seq.tryPick (fun arg \-> if arg.Contains("\--restoreDir=") then Some arg else None)

|> Option.map (fun arg \-> arg.Replace("\--restoreDir=", ""))

let dbArg \=

args

|> Seq.tryPick (fun arg \-> if arg.Contains("\--database=") then Some arg else None)

|> Option.map (fun arg \-> arg.Replace("\--database=", ""))

let restoreDir \=

match restoreTo with

| None \-> Directory.GetCurrentDirectory()

| Some dir \-> Path.GetFullPath(dir)

let dburl \=

match dbArg with

| None \-> "mongodb://localhost:27017"

| Some url \-> url

if not (Directory.Exists(restoreDir)) then

raise (exn "Couldn't set Current Directory")

let dir \= DirectoryInfo(restoreDir)

// we'll use the same backup entry we used before,

// the only difference is that this includes the \_id field

type BackupEntry \= { \_id: ObjectId; directory: string; backupAt: DateTime; filenames: string array; entryType: string }

type EntryType \=

| Attempt

| Success

member this.AsString() \=

match this with

| Attempt \-> "Attempt"

| Success \-> "Success"

static member FromString(value: string) \=

match value with

| "Attempt" \-> Attempt

| "Success" \-> Success

| \_ \-> failwith "Unknown value"

let client \= MongoClient(dburl)

let backupsdb \= client.GetDatabase("backups")

let bucket \= GridFSBucket(backupsdb)

let findBackupsCmd \=

// here we do a find query using an anonymous object

// and we'll filter by successful backup entries

find "backups" {

filter {| entryType \= EntryType.Success.AsString() |}

}

let backups \= backupsdb.RunCommand<FindResult<BackupEntry\>>(JsonCommand findBackupsCmd)

Console.Clear()

AnsiConsole.MarkupLine("\[bold\]Found the following backups\[/\]")

let tbl \= Table()

tbl

.AddColumn("Directory")

.AddColumn("Backup Date")

.AddColumn("Files")

.AddColumn("Entry Type")

let found \= backups.cursor.firstBatch |> Seq.sortByDescending(fun entry \-> entry.backupAt)

found

|> Seq.iteri(fun index entry \->

let date \= $"{entry.backupAt.ToShortDateString()} - {entry.backupAt.ToShortTimeString()}"

let files \= $"{entry.filenames |> Array.length}"

tbl.AddRow(\[| $"{index + 1} - {entry.directory}"; date; files; entry.entryType |\]) |> ignore

)

AnsiConsole.Render(tbl)

let prompt \=

let txt \= TextPrompt<int\>("Select the desired backup")

found

|> Seq.iteri(fun index backup \->

txt.AddChoice<int\>(index + 1) |> ignore)

txt

.DefaultValue(1)

// let's try to also validate that the answer

// is within range of our existing backups

.Validator <- fun value \->

match value with

| value when value <= 0 || value \>= (found |> Seq.length) \-> ValidationResult.Error("The selected backup was not found")

| result \-> ValidationResult.Success()

txt

let response \=

(AnsiConsole

.Prompt<int\>(prompt)) \- 1

let selected \= backups.cursor.firstBatch |> Seq.item response

let filenames \= selected.filenames

AnsiConsole

.Progress()

.Columns(\[|

new SpinnerColumn()

new PercentageColumn()

new ProgressBarColumn()

new TaskDescriptionColumn()

|\])

.Start(fun ctx \->

let settings \= ProgressTaskSettings()

settings.MaxValue <- filenames |> Seq.length |> float

let task \= ctx.AddTask($"Restoring files to %s{dir.FullName}", settings)

for filename in filenames do

// for every entry in our backup we create a file stream

// thankfully MongoDB.Driver.GridFS includes a way to download

// the files directly into a stream, so the process is quite seamless

use filestr \= File.Create(Path.Combine(dir.FullName, filename))

bucket.DownloadToStreamByName(filename, filestr)

task.Increment(1.0)

task.StopTask()

)

AnsiConsole.WriteLine($"Restored: %i{filenames |> Seq.length} files")

that's all we need to start backing up either the current directory or a directory we specify with our arguments, that should look like the following Gif

[![Backup Files](https://res.cloudinary.com/practicaldev/image/fetch/s--VDND9rWp--/c_limit%2Cf_auto%2Cfl_progressive%2Cq_66%2Cw_880/https://dev-to-uploads.s3.amazonaws.com/i/6ol1ynkt7lb0zem4wra9.gif)](https://res.cloudinary.com/practicaldev/image/fetch/s--VDND9rWp--/c_limit%2Cf_auto%2Cfl_progressive%2Cq_66%2Cw_880/https://dev-to-uploads.s3.amazonaws.com/i/6ol1ynkt7lb0zem4wra9.gif)

Now Let's continue with the restores

#!/usr/bin/env \-S dotnet fsi

#r "nuget: MongoDB.Driver"

#r "nuget: MongoDB.Driver.GridFS"

#r "nuget: Mondocks"

#r "nuget: Spectre.Console"

open System

open System.IO

open Spectre.Console

open MongoDB.Driver

open MongoDB.Driver.GridFS

open Mondocks.Types

open Mondocks.Queries

let args \= (fsi.CommandLineArgs)

// first we'll parse some arguments

// granted this is not the best approach but you get the idea

let workDirArg \=

args

|> Seq.tryPick (fun arg \-> if arg.Contains("\--workdir=") then Some arg else None)

|> Option.map (fun arg \-> arg.Replace("\--workdir=", ""))

let dbArg \=

args

|> Seq.tryPick (fun arg \-> if arg.Contains("\--database=") then Some arg else None)

|> Option.map (fun arg \-> arg.Replace("\--database=", ""))

let workdir \=

match workDirArg with

| None \-> Directory.GetCurrentDirectory()

| Some dir \-> Path.GetFullPath(dir)

let dburl \=

match dbArg with

| None \-> "mongodb://localhost:27017"

| Some url \-> url

if not (Directory.Exists(workdir)) then

raise (exn "Couldn't set Current Directory")

let dir \= DirectoryInfo(workdir)

let files \= dir.EnumerateFiles()

let filesTbl \= Table()

filesTbl.AddColumn($"Files - \[bold green\]{dir.FullName}\[/\] :file\_folder:")

filesTbl.AddColumn("Created At :alarm\_clock:")

// add the rows to the table

files

|> Seq.iteri (fun i file \->

let created \= $"{file.CreationTime.ToShortDateString()} {file.CreationTime.ToShortTimeString()}"

filesTbl.AddRow($"\[bold #d78700\]{i + 1}\[/\] - \[dim #d78700\]{file.Name}\[/\]", created) |> ignore

)

AnsiConsole.Render(filesTbl)

/// we could also add a flag at the arguments to backup the whole directory instead of asking

let question \= TextPrompt<string\>("Please enter the files you want to backup (e.g. 0, 1 or all)")

question.AllowEmpty <- true

question.DefaultValue("all")

question.Validator <-

fun text \->

if text \= "all" then

ValidationResult.Success()

else

// let's ensure every value that was typed as a result is actually an integer value

// if there's a value that is not an integer request the answer again

let allInts \=

text.Split(',')

|> Seq.forall(fun i \->

let (parsed, \_) \= Int32.TryParse(i.Trim())

parsed)

if allInts then ValidationResult.Success()

else ValidationResult.Error("the answer must be a comma separated index values string or \\"all\\"")

let ans \= AnsiConsole.Prompt(question)

type BackupType \=

| All

| Specific of int seq

static member FromString(value: string) \=

match value.ToLowerInvariant() with

| "all" \-> All

| sequence \->

sequence.Split(',')

|> Seq.map (fun i \-> Int32.Parse i |> (-) 1)

|> Specific

let toBackup \= BackupType.FromString ans

let filesToBackup \=

// select the files to backup either we go for specific ones or every file

match toBackup with

| All \->

files

| Specific indexes \->

indexes

|> Seq.map(fun index \-> files |> Seq.item index)

// let's create a record that matches up what we'll put in our database

type BackupEntry \= { directory: string; backupAt: DateTime; filenames: string array; entryType: string }

type EntryType \=

| Attempt

| Success

member this.AsString() \=

match this with

| Attempt \-> "Attempt"

| Success \-> "Success"

let entry \=

{ directory \= dir.FullName

backupAt \= DateTime.Now

filenames \=

// we'll use the file name later on

// to retrieve the files so we should ensure

// we save them in some place (in this case MongoDB)

filesToBackup

|> Seq.map(fun f \-> f.Name)

|> Array.ofSeq

entryType \= EntryType.Attempt.AsString() }

let saveCmd \=

// here we use the Mondocks library

// it's a simple insert command using the collection name

// and passing the entry record, by the way, this will create a json string

insert "backups" {

documents \[ entry \]

}

// once we're ready let's contact our database and create a GridFS Bucket

let client \= MongoClient(dburl)

let backupsdb \= client.GetDatabase("backups")

let bucket \= GridFSBucket(backupsdb)

// save the attempt in case we the operation fails we at least know what we wanted to save

backupsdb.RunCommand<InsertResult\>(JsonCommand saveCmd) |> ignore

AnsiConsole

.Progress()

.Columns(\[|

new SpinnerColumn()

new PercentageColumn()

new ProgressBarColumn()

new TaskDescriptionColumn()

|\])

.Start(fun ctx \->

let settings \= ProgressTaskSettings()

settings.MaxValue <- filesToBackup |> Seq.length |> float

let task \= ctx.AddTask($"Saving to database", settings)

// backup every single file to mongodb

for file in filesToBackup do

let id \= bucket.UploadFromStream(file.Name, file.OpenRead())

AnsiConsole.MarkupLine($"\[bold #5f5fff\]FileId:\[/\] \[yellow\]%s{id.ToString()}\[/\]")

task.Increment(1.0)

task.StopTask()

)

// once we are sure we succesfully

// backed everything up we just need to update

// the "attempt" to a "success" and we update the date as well

let savedCmd \=

update "backups" {

updates

\[ { q \= {| directory \= dir.FullName |}

u \= { entry with entryType \= EntryType.Success.AsString(); backupAt \= DateTime.Now }

multi \= Some false

upsert \= Some false

collation \= None

arrayFilters \= None

hint \= None }

\]

}

backupsdb.RunCommand<UpdateResult\>(JsonCommand savedCmd) |> ignore

AnsiConsole.MarkupLine($"\[green\]Backed up %i{filesToBackup |> Seq.length} files\[/\]")

The restoration process was hopefully as simple as the backup and should look like this  
![Restore Backup](https://res.cloudinary.com/practicaldev/image/fetch/s--mXHyDWh8--/c_limit%2Cf_auto%2Cfl_progressive%2Cq_66%2Cw_880/https://dev-to-uploads.s3.amazonaws.com/i/0l2k9as2pm2wdr3bg1so.gif)

And that's it! Hopefully, I showed you a bit of the F#'s scripting capabilities as well as how simple is to use Mondocks without sacrificing the MongoDB driver, since it's a side by side usage thing here's the project if you'd like to take a look

## ![GitHub logo](https://res.cloudinary.com/practicaldev/image/fetch/s--i3JOwpme--/c_limit%2Cf_auto%2Cfl_progressive%2Cq_auto%2Cw_880/https://dev.to/assets/github-logo-ba8488d21cd8ee1fee097b8410db9deaa41d0ca30b004c0c63de0a479114156f.svg) [ AngelMunoz ](https://github.com/AngelMunoz) / [Mondocks](https://github.com/AngelMunoz/Mondocks) 

### An alternative way to interact with MongoDB databases from F# that allows you to use mongo-idiomatic constructs

[![nuget](https://camo.githubusercontent.com/6d39d0acf21c518678f153ff086b813ec5ec039ea081e9c7c69dda4da03a125a/68747470733a2f2f62616467656e2e6e65742f6e756765742f762f6d6f6e646f636b73)](https://camo.githubusercontent.com/6d39d0acf21c518678f153ff086b813ec5ec039ea081e9c7c69dda4da03a125a/68747470733a2f2f62616467656e2e6e65742f6e756765742f762f6d6f6e646f636b73) [![Binder](https://camo.githubusercontent.com/eec63e2390fc0954434176e755039588d957ad7919337c441175a94802b53649/68747470733a2f2f6e6f7465626f6f6b732e67657369732e6f72672f62696e6465722f62616467655f6c6f676f2e737667)](https://notebooks.gesis.org/binder/v2/gh/AngelMunoz/Mondocks/HEAD)

> ```
> dotnet add package Mondocks.Net
> # or for fable/nodejs
> dotnet add package Mondocks.Fable
> ```

> This library is based on the mongodb extended json spec and mongodb manual reference
> 
> [https://docs.mongodb.com/manual/reference/mongodb-extended-json/](https://docs.mongodb.com/manual/reference/mongodb-extended-json/) > [https://docs.mongodb.com/manual/reference/command/](https://docs.mongodb.com/manual/reference/command/)

This library provides a set of familiar tools if you work with mongo databases and can be a step into more F# goodies, it doesn't prevent you from using the usual MongoDB/.NET driver so you can use them side by side. It also can help you if you have a lot of flexible data inside your database as oposed to the usual strict schemas that F#/C# are used to from SQL tools, this provides a DSL that allow you to create `MongoDB Commands (raw queries)` leveraging the dynamism of anonymous records since they behave almost like javascript objects Writing commands should be almost painless these commands produce a JSON string that can be utilized directly on your application or even…

Also, Shout out to [Spectre.Console](https://spectresystems.github.io/spectre.console/) for the amazing console output.

Remember that F# is cross-platform, so if you want to run these scripts on Linux/MacOS as well you should be able to do so. F# is powerful yet it won't get in your way, it will most likely help you figure out how to do things nicely.

If you have further comments or doubts, please let me know down below or ping me on Twitter 😁