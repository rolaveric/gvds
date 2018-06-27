# Design doc

**WORK IN PROGRESS!!**

Purpose of this doc is to provide as much detail as possible about how this service should be built, before it's built.  
Hopefully that will convey the "method to the madness" for any future users or contributors about why gvds is built the way it is.

## Aims

Taken from `README.md`.  These should act as the cornerstone for how the system is structured.

* __Be performant__: The quicker the feedback loop, the happier everyone is.
* __Be scalable__: Today it's your landing page; Tomorrow it's 20 SPAs with 30 screenshots, each, duplicated across phone, tablet, laptop, and desktop screen sizes.
* __Be modular__: S3 vs Filesystem; GRPc vs Thrift; ImageMagick vs pixelmatch; Containers and Docker vs processes and ipc; Everyone's going to have different techs that suit them, and you should be able to change your mind later with minimal pain.

## System components

TODO: Get a real image drawn up

```
                          /--> [ API ]
[ Service / Controller ] <--> [ Storage ]            /--> [ Worker ]
                          \--> [ Job Orchestrator ] <--> [ Worker ]
                                                     \--> [ Worker ]
```

__API__  
Marshalls and unmarshalls requests and responses for endpoints along the specified transport(s).

__Service__  
Provides service endpoints and coordinates between the other components.

__Storage__  
Handles persisting images (run screenshots, baselines, and diffs) and metadata (projects, runs, image URLs, etc.)

__Job Orchestrator__  
Takes jobs (2 images to compare, plus options) and allocates them to tracked workers, then passes results back to Service to be persisted.

__Worker__  
Takes 2 images and options (eg. thresholds, areas to ignore, etc.) and returns a diff image and metadata (eg. diff percentage)

## API

Requests will comes from 4 main sources:
* __UI__: Web portal to display results and allow baselines to be reset.
* __Clients__: Selenium/E2E test runners starting "runs" and POSTing screenshots.
* __CD__: Deployment pipeline tools looking to block deployments until visual regression test failures have been reviewed.
* __Sysadmin/Infra__: eg. Add/remove workers, health checks, storage truncation (only really need baseline + last failure for each test), etc.

TODO: List out endpoints, using source as prefix. eg. ui_getProjectsSummary(): ProjectsSummary  
TODO: Create Swagger/Open API definition

## Security

TODO While most will use service internally, with little/no use for security, it's worth adding it to the designs now rather than trying to shoehorn it later

## Storage

The storage module needs to handle 3 main object types:
* Image file
* Project
* Project list

So it's interface should be:
* GetImage(url: URL): Byte[]
* DeleteImages(urls: URL[])
* StoreImage(image: Byte[]): URL
* GetProject(projectName: string): Project{}
* PutProject(projectName: string, project: Project{})
* DeleteProject(projectName: string)
* GetProjectList(): string[]
* AddProjectToList(projectName: string)
* RemoveProjectFromList(projectName: string)

## Job Orchestrator

TODO Discuss provisioning workers vs tracking workers (ie. Being told, via admin channel, of a worker's existence)  
TODO Circuit breaker for crashing workers  
TODO Response to Service vs direct calls to Storage

## Worker

TODO Request/Response interface  
TODO Different transports. eg. HTTP vs ipc
