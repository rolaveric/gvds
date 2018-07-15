# Design doc

**WORK IN PROGRESS!!**

Purpose of this doc is to provide as much detail as possible about how this service should be built, before it's built.  
Hopefully that will convey the "method to the madness" for any future users or contributors about why gvds is built the way it is.

## Aims

Taken from `README.md`.  These should act as the cornerstone for how the system is structured.

* __Be performant__: The quicker the feedback loop, the happier everyone is.
* __Be scalable__: Today it's your landing page; Tomorrow it's 20 SPAs with 30 screenshots, each, duplicated across phone, tablet, laptop, and desktop screen sizes.
* __Be modular__: S3 vs Filesystem; GRPc vs Thrift; ImageMagick vs pixelmatch; Containers and Docker vs processes and ipc; Everyone's going to have different techs that suit them, and you should be able to change your mind later with minimal pain.

## Data

There is a hiearchy of `projects -> suites -> runs -> tests`, shamelessly taken from the [Spectre](https://github.com/wearefriday/spectre) visual regression testing tool which GVDS hopes to expand on.

Images are referenced as URIs, which is useful if they end up stored in an S3 CDN type environment.

Jobs are also persisted in Storage, which allows them to be picked up again if a Worker starts one and then crashes for some reason.

## System components

TODO: Get a real image drawn up

```
Frontend          Service Layer         Backend         Worker pool
[ UI API ]     -> [ UI Service ]     -> /---------\
[ Client API ] -> [ Client Service ] -> | Storage |  <- [ Job Orchestrator ] <- [ Worker ]
[ CD API ]     -> [ CD Service ]     -> |         |
[ Admin API ]  -> [ Admin Service ]  -> \---------/
```

* __APIs__: Marshalls and unmarshalls requests and responses for endpoints along the specified transport(s).
* __Services__: Application logic that's exposed through endpoints that ultimately interact with the storage layer.
* __Storage__: Data persistance layer, including job queues, image artifacts, and project/suite/run/test metadata.
* __Job Orchestrator__: Handles tracking workers and assigning work to them. Also handles inserting results into Storage (if workers don't support doing so themselves).
* __Worker__: Takes 2 images and options (eg. thresholds, areas to ignore, etc.), generates a diff (both image and metadata), and generates thumbnails for new images.

### Communication channels

- **Client-Frontend**: Any network transport you choose to support. eg. HTTPS, thrift, GRPc, etc.
- **Storage-Database**: Dependant on the database. Ideally something with a native Go driver.
- **Orchestrator-Workers**: If Workers are Go routines, then it would be channels. If they're separate processes on the local machine, they could communicate via ipc. Otherwise it will be a network transport on a port separate to the frontend.
- **Workers-Storage**: Storage needs to expose endpoints on a third port for workers to request through.

Everything else is expected to be Go modules which communicate via interfaces.

## Frontend API

Requests will comes from 4 main sources:
* __UI__: Web portal to display results and allow baselines to be reset.
* __Clients__: Selenium/E2E test runners starting "runs" and POSTing screenshots.
* __CD__: Deployment pipeline tools looking to block deployments until visual regression test failures have been reviewed.
* __Admin__: eg. Add/remove workers, health checks, storage truncation (only really need baseline + last failure for each test), etc.

Each of these sources is assigned it's own service to provide endpoints to that source.

### UI endpoints

* __ui_getProjectsSummary() ProjectsSummary__: Returns all project metadata, along with URIs for images.
* __ui_getImage(uri: string): Byte[]__: Returns an image for a given URI.
* __ui_setBaseline(testId: string, runId: string): void__: Sets the image uploaded for a given test+run as the new baseline for that test.
* __ui_deleteSuite(suiteId: string): void__: Deletes a suite of test results.

> Note: This is a minimal list of endpoints. A full API would include more granular GET endpoints to avoid downloading the WHOLE projects graph.

TODO: Define `ProjectsSummary` type
TODO: Create Swagger/Open API definitions

### Client endpoints

* __client_createRun(projectId: string, suiteId: string) string__: Creates a new test run, returning an ID to upload test screenshots against.
* __client_uploadTest(runId: string, testName: string, image: Byte[], options: {}) void__: Uploads an image for a test run, with a given name. May also includes options to be used when testing the image - such as dead zones to ignore and thresholds that need to be broken before the test is considered "broken". Internally creates a job which will be picked up and handled asynchronously.

TODO: Create Swagger/Open API definitions

### CD endpoints

* __cd_getSuiteStatus(suitId: string) SuiteStatus__: Returns the status of a given suite based on it's last run: PASS, FAIL, or PENDING. It will return PENDING if there is at least 1 job still pending for the suite.

TODO: Define `SuiteStatus` type
TODO: Create Swagger/Open API definitions

### Admin endpoints

* __admin_getHealth() HealthSummary__: Returns a health summary of the application.
* __admin_getMetrics() MetricsSummary__: Returns metrics from the application, such as those produced by Prometheus.
* __admin_getWorkers() WorkerSummary[]__: Returns details about the known worker instances.
* __admin_addWorker(worker: WorkerSummary) error__: Starts tracking a given worker, assigning it jobs as they become available.
* __admin_removeWorker(workerId: string) error__: Stops using the given worker for jobs. Blocks until the worker becomes idle.

> Note: I suspect this will be an incomplete list of endpoints.

TODO: Create Swagger/Open API definitions

## Security

Security will be handled as middleware on the frontend API, standing between the API endpoint and the service layer.  
Aside from being API middleware, no other assumptions will be made about the security implementation.

To start with, only 2 security implementations will be provided:
* NoopSecurity: Do nothing. Depend on other network features to provide security.
* StaticBearerSecurity: Require requests to include a `Bearer` authentication token, which will be statically set through environment variables. A different token can be set for each service, and can be turned off completely for some  
eg. UI could allow anon access, while Client requires a "ICanUpload" token, and Admin requires a "TooSecretForYou" token.

## Storage

The storage module can be implemented in anyway that fits the user's requirements or infrastructure, as long as it implements the following API and exposes it in a way that can be accessed by the service layer and worker pool (eg. Go module).

TODO List procedure calls that it needs to support.  
TODO Include details about enabling worker and orchestration access

Possible implementations:

* File system: Files in folders, single JSON or XML file with projects data.
* SQL: Use Go's `database/sql` module with any SQL implementation with a supported driver.
* NoSQL: MongoDB or CouchDB for projects data, and an S3 implementation for images.

Starting implementation will be file system based.

## Job Orchestrator

The job orchestrator is responsible for *tracking* workers, not provisioning them.  Provisioning workers needs to be done externally, with details passed in via the Admin module to add them to the pool.  Workers can also be deactivated by the Admin module.

Jobs are assigned to workers by providing them a job ID. The workers themselves are then expected to retrieve the job data from the storage module. When finished, they pass the result to storage themselves, then notify the orchestrator that they're available to pickup another job.

A worker is considered "working", and therefore unavailable, once it's been given a job. Only once a "done" message is returned does it get returned to the available pool to receive another job.

If a job takes too long, the worker is considered unresponsive and remove from the pool, and the job is reassigned to another available worker.

The job orchestrator should gracefully handle when there are no valid workers available (jobs are simply queued up in storage).

When the job orchestrator first starts up, it retrieves the list of available jobs from Storage and holds them in memory.  It should also periodically check Storage for any jobs that might have been missed or lost (eg. Once every 5 minutes or so), and should make a log entry if it does find one - in case this is a systemic issue that needs resolving.

## Worker

Workers are assigned Job IDs, which can then be used to query Storage for the relevant data (ie. screenshots), and to write the results back to Storage.

Workers need to handle one (or both) of two operations: thumbnail generation, and image diffing.

TODO Request/Response interface  
TODO Different transports. eg. HTTP vs ipc

## Scaling

- (likely) Jobs are not actioned fast enough -> provision more workers
- (possible) UI does not respond fast enough due to large metadata set -> UI must request and render small subsets of the data at a time.
- (possible) UI does not respond fast enough due to size/number of images flowing through it -> expose images on a CDN that's accessed directly by the client, rather than needing to be piped through the rest of the application.
- (unlikely) too many jobs/workers for Orchestrator to manage -> allow for duplicate Orchestrators on a round robin
- (unlikely) too many requests coming through the front-end -> duplicate system in front of a shared/synced storage system

The easiest measure for scaling is to duplicate the whole system across project/team boundaries. There's no reason to keep everything centralised except that it's convenient to have a single endpoint to communicate with and a single instance to manage.

## Potential features

Some notes on potential feature enhancements.

- **Elastic workers**: Job Orchestrator would need to know how to provision and shutdown (not just orphan) workers, or at least know where and how to send requests for more workers to be spun up.
- **Storage sharding**: This would mostly be an implementation detail of the database. The Storage API _may_ need to be aware of this feature too.
- **Clustering**: The trick with multiple instances is that you would need to avoid 2+ orchestrators assigning out the same jobs. The occassional cross-over should be harmless (they should both produce the same result), just a waste of resources. Otherwise it's down to how well the database implementation can scale with multiple application instances calling it.
