# Search Engine

A small full-text search engine written in **C# (.NET 5)**. It builds an **inverted index** of text documents (for each word, the list of documents that contain it) and finds documents with simple queries using **AND**, **OR** and **NOT**.

The index is saved with **Entity Framework Core**: in memory by default, or in **SQL Server** if you set a connection string.

The search engine can be used in three ways:

| Project | What it is |
|---|---|
| `ConsoleApp` | Index a folder of text files and search it in the terminal |
| `WebApi` | A REST API (with Swagger) to add documents, search, get word hints and clear the index |
| `WebMVC` | A search web page with live word hints and a results page, plus the same API |

## How it works

**Indexing a document**

1. The text is split into words. A word starts with a letter and can contain letters and digits (`covid19` is one word, `2021` is ignored). Everything is made lowercase.
2. Common **stop words** are removed (`a`, `and`, `is`, `this`, `to`, `with`, … the full list is in `SearchEngine/StringUtils.cs`). Note: `the` is **not** in the list.
3. Each word is **stemmed** with the Porter2 stemmer, so `doctor` and `doctors` both become `doctor`, and `medicine` becomes `medicin`.
4. Each stemmed word is linked to the document name.

**Searching**

Write words separated by spaces:

| You write | Meaning |
|---|---|
| `word` | The document **must** contain this word. All such words must be in the document (AND). |
| `+word` | **Add** documents that contain this word (OR). |
| `-word` | **Remove** documents that contain this word (NOT). |

The steps run in this order: first the plain words (AND), then the `+` words are **added** to the result, then the `-` words are removed. Query words are stemmed the same way, so `doctors` also finds `doctor`.

Examples, using the sample files in `TestProject1/TestDocs/testDocs`:

| Query | Result |
|---|---|
| `doctor` | `59637.txt`, `59639.txt`, `59641.txt`, `59646.txt`, `59652.txt` |
| `Cause +People -pretty` | `59631.txt`, `59635.txt`, `59639.txt`, `59648.txt` |
| `+Cause +People +pretty` | `59631.txt`, `59632.txt`, `59633.txt`, `59635.txt`, `59637.txt`, `59639.txt`, `59648.txt` |
| `Cause People pretty` | no documents |

Good to know:
- A query with **only** `-` words always returns nothing.
- A query with only stop words (for example `this`) returns nothing.
- Punctuation in the query is not removed: search `hello`, not `hello,`.

## Requirements

- **.NET 5 SDK.** All projects target `net5.0`. .NET 5 is no longer supported by Microsoft, but you can still download it from the .NET website. A newer SDK also works if the **.NET 5 runtime** (ASP.NET Core 5 runtime for the web projects) is installed too.
- Optional: **SQL Server**, only if you want to save the index in a database (see [Using SQL Server](#using-sql-server-optional)).
- Visual Studio 2019, JetBrains Rider or just the `dotnet` command line. Open `EFCore.sln` in an IDE.

## How to run

Run all commands from the project folder (the one with `EFCore.sln`).

### 1. Console app

Give the folder to index as the argument:

```bash
dotnet run --project ConsoleApp -- TestProject1/TestDocs/testDocs
```

The app reads every file in the folder and its sub-folders. It prints `using In Memory`, then `adding <file>` for each file. When it prints `start engine`, type a query and press Enter. It prints the names of the matching files (relative to the folder), or `no doc found`. Type `exit` to quit.

```
start engine
Cause +People -pretty
59631.txt
59635.txt
59639.txt
59648.txt
exit
end engine
```

Without a folder argument, the app starts with an empty index (unless you use SQL Server, which keeps the index from earlier runs).

### 2. Web API

```bash
dotnet dev-certs https --trust
dotnet run --project WebApi --launch-profile WebApi
```

Open **<https://localhost:5001/swagger>** to try the endpoints. The index starts empty each time the API starts, so add documents first.

| Method | Address | What it does |
|---|---|---|
| POST | `/Search/PostDocuments` | Adds documents. Body: a JSON object of `"document name": "document text"` |
| GET | `/Search/Query?query=...` | Returns the names of the matching documents |
| GET | `/Search/GetHints?hint=...` | Returns indexed words that start with the given text |
| DELETE | `/Search/ClearIndex` | Removes everything from the index |

Example with `curl`:

```bash
curl -k -X POST https://localhost:5001/Search/PostDocuments \
     -H "Content-Type: application/json" \
     -d '{"doc1": "My doctor gave me new medicine", "doc2": "The doctors were busy"}'

curl -k "https://localhost:5001/Search/Query?query=doctor%20-medicine"
# ["doc2"]
```

(`-k` skips the certificate check. `%20` is a space. These commands are for macOS, Linux or Git Bash. In the Windows Command Prompt the quotes work differently, so it's easier to use the Swagger page.)

### 3. Web page (WebMVC)

```bash
dotnet dev-certs https --trust
dotnet run --project WebMVC --launch-profile WebMVC
```

The **index starts empty** each time the app starts, and the web page has no upload form. Add documents first with the API above (`POST https://localhost:5001/Search/PostDocuments`, for example with the `curl` command). WebMVC has the same `/Search/...` API as WebApi, but no Swagger page.

Then open the search page: **<https://localhost:5001/Main/index.html>**

- Type a query. When the last word has 3 or more letters, the page shows up to 11 **hints**: indexed words that start with it. Hints are stemmed words (for example `medicin`), because that's how words are stored.
- Click **Search**. The results page lists the matching document names, or shows *No Doc Found* with a different background video.
- Open the menu on the results page and click **Back To Search Tab** to search again.

The plain address `https://localhost:5001/` only shows the default ASP.NET "Welcome" page.

**Note:** WebApi and WebMVC both use port **5001**, so run only one of them at a time. The search page's hint requests and the "Back To Search Tab" link are hard-coded to `https://localhost:5001`, so WebMVC must run on this port.

## Using SQL Server (optional)

The apps read a setting called `ConnectionString` with `System.Configuration.ConfigurationManager`. No config file with this setting is in the repo, so by default they use an **in-memory** database, and the index is lost when the app stops.

To use SQL Server, add an `App.config` file to the project you run (for example `ConsoleApp/App.config`):

```xml
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <appSettings>
    <add key="ConnectionString" value="Server=.;Database=InvertedIndex;Trusted_Connection=True;" />
  </appSettings>
</configuration>
```

Change the connection string to match your SQL Server. The database and its tables are created automatically on first use. `ClearIndex` **deletes the whole database** and creates it again.

## Tests

```bash
dotnet test
```

The tests (xUnit with NSubstitute) are in `TestProject1`. They cover the tokenizer, stop words and stemming, query parsing, the file reader, the inverted index (with the sample files in `TestProject1/TestDocs`) and the console search loop. They use an in-memory database, so no SQL Server is needed.

**Note:** I could not build or run the C# code while writing this README, because the .NET package server (NuGet) was not reachable. The steps above come from reading the code. I checked the search logic by re-writing it in Python and running it on the sample files: the results match the expected results in the tests and the examples above.

## Project structure

| Folder | What it is |
|---|---|
| `SearchEngine/` | The core library: `Tokenizer`, `StringUtils` (stop words, stemming), `UserInput` (query parsing), `InvertedIndex`, `InvertedIndexMap` and `InvertedIndexContext` (EF Core storage), `FileReader`, and the console search loop (`SearchEngine.cs`) |
| `ConsoleApp/` | Terminal app (`Manger.cs` holds the `Manager` class) |
| `WebApi/` | REST API with Swagger |
| `WebMVC/` | Web app: the search page is in `wwwroot/Main/`, the results page is `Views/ChangePage/Query.cshtml` |
| `TestProject1/` | Unit tests and sample text files (`TestDocs/`) |
| `EFCore.sln` | Solution file with all five projects |

## Tech

- C#, .NET 5, ASP.NET Core (Web API and MVC)
- Entity Framework Core 5 (in-memory and SQL Server providers)
- Porter2Stemmer
- xUnit, NSubstitute
- Bootstrap and jQuery on the search page
