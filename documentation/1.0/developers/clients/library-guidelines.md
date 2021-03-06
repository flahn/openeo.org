# Client library development guidelines

This is a proposal for workflows that client libraries should support to make the experience with each library similar and users can easily adopt examples and workflows.

For best experience libraries should still embrace best practices common in their environments. This means clients can...

* choose which kind of casing they use (see below). 
* feel free to implement aliases for methods.

## Conventions

### Casing

Clients can use `snake_case`, `camelCase` or any method used commonly in their environment. For example, the API request to get a list of collections can either be names `get_collections` or `getCollections`. This applies for all names, including scopes, method names and parameters.

### Scopes

Each method belongs to a scope. To achieve this in object-oriented (OO) programming languages, methods would be part of a class. If programming languages don't support scopes, you may need to simulate it somehow to prevent name collisions, e.g. by adding a prefix to the method names (like in the "procedural style" example below). Best practices for this will likely evolve over time.

Example for the `clientVersion` method in `openEO`:

* Procedural style: `openeo_client_version()` 
* Object-oriented style:
  ```java
  OpenEO obj = new OpenEO();
  obj.clientVersion();
  ```

If you can't store scope data in an object, you may need to pass these information as argument(s) to the method.

Example:

* Procedural style:
  ```php
  $connection = openeo_connect("https://openeo.org");
  openeo_capabilities($connection);
  ```
* Object-oriented style:
  ```java
  OpenEO obj = new OpenEO();
  Connection con = obj.connect("https://openeo.org");
  con.capabilities();
  ```

### Scope categories

Each scope is assigned to a scope category, of which there are three:

* *Root* category: Contains only the scope `openEO`.
* *API* category: Mostly methods hiding API calls to the back-ends. Methods may be implemented asynchronously. Contains the scopes `Connection`, `File`, `Job`, `UserProcess`, `Service`.
* *Content*: Mostly methods hiding the complexity of response content. Methods are usually implemented synchronously. Currently contains only the scope `Capabilities`. Method names should be prefixed if name collisions are likely.

Method names across ALL the scopes that belong to the *root* or *API* categories MUST be unique. This is the case because the parameter in `hasFeature(method_name)` must be unambiguous.

Method names of scopes in the *Content* category may collide with method names of scopes in the *root*/*API* categories and names should be prefixed if collisions of names between different scope categories are to be expected.

### Parameters

The parameters usually follow the request schemes in the openAPI specification. The parameters should follow their characteristics, for example regarding the default values.

Some methods have a long list of (optional) parameters. This is easy to implement in languages that support named parameters such as R. For example, creating a job in R with a budget would lead to this method call:

```R
createJob(process_graph = {...}, budget = 123)
```

Other languages that only support non-named parameters (i.e. the order of parameters is fixed) need to fill many parameters with default values, which is not convenient for a user. The example above in PHP would be:

```php
createJob({...}, null, null, null, null, null, 123)
```

To avoid such method calls client developers should consider to pass either

* an instance of a class, which contains all parameters as member variables or
* the required parameters directly and the optional parameters as a dictionary (see example below).

This basically emulates named parameters. The member variables / dictionary keys should use the same names as the parameters. The exemplary method call in PHP could be improved as follows:

```php
createJob({...}, [budget => 123])
```

## Method mappings

**Note:** Some scopes for response JSON objects are still missing. We are open for proposals.

Parameters with a leading `?` are optional.

### Scope: `openEO` (root category)

| Description                                                  | Client method |
| ------------------------------------------------------------ | ------------- |
| Connect to a back-end, includes version discovery (`GET /.well-known/openeo`), requesting capabilities and authentication where required. Returns `Connection`. | `connect(url, ?authType, ?authOptions)` |
| Get client library version.                                  | `clientVersion()` |

#### Parameters

* **authType** in `connect`: `null`, `basic` or `oidc` (non-exclusive). Defaults to `null` (no authentication).
* **authOptions** in `connect`: May hold additional data for authentication, for example a username and password for `basic` authentication.

### Scope: `Connection` (API category)

| Description                                                   | API Request                                                | Client method |
| ------------------------------------------------------------- | ---------------------------------------------------------- | ------------- |
| Get the capabilities of the back-end. Returns `Capabilities`. | `GET /`                                                    | `capabilities()` |
| List the supported output file formats.                       | `GET /file_formats`                                        | `listFileTypes()` |
| List the supported secondary service types.                   | `GET /service_types`                                       | `listServiceTypes()` |
| List the supported UDF runtimes.                              | `GET /udf_runtimes`                                        | `listUdfRuntimes()` |
| List all collections available on the back-end.               | `GET /collections`                                         | `listCollections()` |
| Get information about a single collection.                    | `GET /collections/{collection_id}`                         | `describeCollection(collection_id)` |
| List all pre-defined processes available on the back-end.     | `GET /processes`                                           | `listProcesses()` |
| List authentication providers (may also list HTTP Basic).     | `GET /credentials/oidc` / `GET /credentials/basic`         | `listAuthProviders()` |
| Get information about the authenticated user.                 | `GET /me`                                                  | `describeAccount()` |
| Lists all files from a user. Returns a list of `File`.        | `GET /files`                                               | `listFiles()` |
| Opens an (existing or non-existing) file without reading it. Returns a `File`. | *None*                                    | `getFile(path)` |
| Upload a user file. Returns a `File`.                         | Shortcut: `getFile(path).uploadFile(source)`               | `uploadFile(source, path)` |
| Validates a process graph.                                    | `POST /validation`                                         | `validateProcess(process)` |
| Lists all user-defined processes of the authenticated user. Returns a list of `UserProcess`. | `GET /process_graphs`       | `listUserProcesses()` |
| Creates a new user-defined process. Returns a `UserProcess`.  | Shortcut: `getUserProcess(id).replaceUserProcess(process)` | `setUserProcess(id, process)` |
| Get all information about a user-defined process. Returns a `UserProcess`. | `GET /process_graphs/{process_graph_id}`      | `getUserProcess(id)` |
| Executes a process graph synchronously.                       | `POST /result`                                             | `computeResult(process, ?plan, ?budget, ?additional)` |
| Lists all jobs of the authenticated user. Returns a list of `Job`. | `GET /jobs`                                           | `listJobs()` |
| Creates a new job. Returns a `Job`.                           | `POST /jobs`                                               | `createJob(process, ?title, ?description, ?plan, ?budget, ?additional)` |
| Get all information about a job. Returns a `Job`.             | `GET /jobs/{job_id}`                                       | `getJob(id)` |
| Lists all secondary services of the authenticated user. Returns a list of `Service`. | `GET /services`                     | `listServices()` |
| Creates a new secondary service. Returns  a `Service`.        | `POST /services`                                           | `createService(process, type, ?title, ?description, ?enabled, ?parameters, ?plan, ?budget, ?additional)` |
| Get all information about a service. Returns a `Service`.     | `GET /services/{service_id}`                               | `getService(id)` |

#### Parameters

* **options** in `authenticateOIDC`: May hold additional data required for OpenID connect authentication.
* **additional** in `createJob` and `createService` (also below in `updateJob` and `updateService`): May hold additional key-value-pairs as object. The object should be merged with the body of the request.

### Scope `Capabilities` (Content category)

Should be prefixed with `Capabilities` if collisions of names between different scope categories are to be expected.

| Description                                      | Field                  | Client method |
| ------------------------------------------------ | ---------------------- | ------------- |
| Get the implemented openEO version.              | `api_version`          | `apiVersion()` |
| Get the back-end version.                        | `backend_version`      | `backendVersion()` |
| Is the back-end suitable for use in production?  | `production`           | `isStable()` |
| Get the name of the back-end.                    | `title`                | `title()` |
| Get the description of the back-end.             | `description`          | `description()` |
| List all supported features / endpoints.         | `endpoints`            | `listFeatures()` |
| Check whether a feature / endpoint is supported. | `endpoints` > ...      | `hasFeature(methodName)` |
| Get the default billing currency.                | `billing` > `currency` | `currency()` |
| List all billing plans.                          | `billing` > `plans`    | `listPlans()` |
| List all links.                                  | `links`                | `links()` |

#### Parameters

* **methodName** in `hasFeature`: The name of a client method in any of the scopes that are part of the *API* category. E.g. `hasFeature("describeAccount")` checks whether the `GET /me` endpoint is contained in the capabilities response's `endpoints` object.

### Scope: `File` (API category)

The `File` scope internally knows the `path`.

| Description           | API Request            | Client method |
| --------------------- | ---------------------- | ------------- |
| Download a user file. | `GET /files/{path}`    | `downloadFile(target)` |
| Upload a user file.   | `PUT /files/{path}`    | `uploadFile(source)` |
| Delete a user file.   | `DELETE /files/{path}` | `deleteFile()` |

#### Parameters

* **target** in `downloadFile`: Path to a local file or folder.

### Scope: `Job` (API category)

The `Job` scope internally knows the `job_id`.

| Description                                | API Request                           | Client method |
| ------------------------------------------ | ------------------------------------- | ------------- |
| Get all job information.                   | `GET /jobs/{job_id}`                  | `describeJob()` |
| Modify a job at the back-end.              | `PATCH /jobs/{job_id}`                | `updateJob(?process, ?title, ?description, ?plan, ?budget, ?additional)` |
| Delete a job                               | `DELETE /jobs/{job_id}`               | `deleteJob()` |
| Calculate an time/cost estimate for a job. | `GET /jobs/{job_id}/estimate`         | `estimateJob()` |
| Get the log files for a job.               | `GET /jobs/{job_id}/logs`             | `debugJob()` |
| Start / queue a job for processing.        | `POST /jobs/{job_id}/results`         | `startJob()` |
| Stop / cancel job processing.              | `DELETE /jobs/{job_id}/results`       | `stopJob()` |
| Get STAC catalog with download links.      | `GET /jobs/{job_id}/results`          | `listResults()` |
| Download job results.                      | `GET /jobs/{job_id}/results` > assets | `downloadResults(target)` |

#### Parameters

* **target** in `downloadResults`: Path to a local folder.

### Scope: `UserProcess` (API category)

The `UserProcess` scope manages the user-defined processes and internally knows the `process_graph_id`.

| Description                                               | API Request                                 | Client method |
| --------------------------------------------------------- | ------------------------------------------- | ------------- |
| Get information about a user-defined process.             | `GET /process_graphs/{process_graph_id}`    | `describeUserProcess()` |
| Create or replace a user-defined process at the back-end. | `PUT /process_graphs/{process_graph_id}`    | `replaceUserProcess(process)` |
| Delete a user-defined process.                            | `DELETE /process_graphs/{process_graph_id}` | `deleteUserProcess()` |

### Scope: `Service` (API category)

The `Service` scope internally knows the `service_id`.

| Description                                        | API Request                       | Client method |
| -------------------------------------------------- | --------------------------------- | ------------- |
| Get all information about a secondary web service. | `GET /services/{service_id}`      | `describeService()` |
| Get the log files for a web service.               | `GET /services/{service_id}/logs` | `debugService()` |
| Modify a secondary web service at the back-end.    | `PATCH /services/{service_id}`    | `updateService(?process, ?title, ?description, ?enabled, ?parameters, ?plan, ?budget, ?additional)` |
| Delete a secondary web service.                    | `DELETE /services/{service_id}`   | `deleteService()` |

## Processes

The processes a back-end supports may be offered by the clients as methods in its own scope. The method names should follow the process names, but the conventions listed above can be applied here as well, e.g. converting `filter_bands` to `filterBands`. As parameters have no natural or technical ordering in the JSON objects, clients must come up with a reasonable ordering of parameters if required. This could be inspired by existing clients. The way of building a process graph from processes heavily depends on the technical capabilities of the programming language. Therefore it may differ between the client libraries. Follow the best practices of the programming language, e.g. support method chaining if possible.

## Workflow example

Some simplified example workflows using different programming styles are listed below. The following steps are executed:

1. Loading the client library.
2. Connecting to a back-end and authenticating with username and password via OpenID Connect.
3. Requesting the capabilities and showing the implemented openEO version of the back-end.
4. Showing information about the "Sentinel-2A" collection.
5. Showing information about all processes supported by the back-end.
6. Building a simple process graph.
7. Creating a job.
8. Pushing the job to the processing queue.
9. After a while, showing the job details, e.g. checking the job status.
10. Once processing is finished, downloading the job results to the local directory `/tmp/job_results/`.

Please note that the examples below do not comply to the latest process specification. They are meant to show the differences in client development, but are no working examples!

### R (functional style)

```r
library(openeo)

con = connect("https://openeo.org", ...)
cap = capabilities()
cap$api_version()
describe_collection("Sentinel-2A")
list_processes()

datacube = datacube()
# Chain processes, implementation is up to the client

job = createJob(process = datacube)
start_job(job = job)
describe_job(job = job)
download_results(job = job, folder = "/tmp/job_results/")
```

### Python (mixed style)

```python
import openeo

con = openeo.connect("https://openeo.org").authenticate_basic("username", "password")
cap = con.capabilities()
print(cap.api_version())
print(con.describe_collection("Sentinel-2A"))
print(con.list_processes())

datacube = con.load_collection(...)
# Chain processes, implementation is up to the client

job = datacube.create_job()
job.start_job()
print(job.describe_job())
job.download_results("/tmp/job_results/")
```

### Java (object oriented style)

```java
import org.openeo.OpenEO;

OpenEO obj = new OpenEO();
Connection con = obj.connect("https://openeo.org", ...);
Capabilities cap = con.capabilities();
System.out.println(cap.apiVersion());
System.out.println(con.describeCollection("Sentinel-2A"));
System.out.println(con.listProcesses());

DataCube cube = con.createDataCube()
// Chain processes, implementation is up to the client

Job job = cube.createJob();
job.startJob();
System.out.println(job.describeJob());
job.downloadResults("/tmp/job_results/");
```

### PHP (procedural style)

```php
require_once("/path/to/openeo.php");

$connection = openeo_connect("http://openeo.org", ...);
$capabilities = openeo_capabilities($connection);
echo openeo_api_version($capabilites);
echo openeo_describe_collection($connection, "Sentinel-2A");
echo openeo_list_processes($connection);

$datacube = openeo_datacube($connection);
// Chain processes, implementation is up to the client

$job = openeo_create_job($connection, $datacube);
openeo_start_job($job);
echo openeo_describe_job($job);
openeo_download_results($job, "/tmp/job_results/");
```
