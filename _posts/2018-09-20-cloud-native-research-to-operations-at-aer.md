---
layout: post
title:  "Cloud-Native Research to Operations at AER"
date:   2018-09-20 08:29:02 -0400
---
Atmospheric and Environmental Research ([AER](https://www.aer.com)) is home to scientists and software engineers specialized in helping governments and businesses solve issues related to weather and climate.  The Amazon Web Services ([AWS](https://aws.amazon.com)) public cloud platform offers an assortment of on-demand and effectively unlimited compute and data storage resources.  This article covers some of AER's methods and strategies for transitioning scientific research to cloud-native operational capabilities on AWS.  We'll discuss how scientists can benefit from running algorithms in [Docker](https://www.docker.com) containers; how software engineers can help deploy and scale these containerized algorithms in the cloud using [AWS Batch](https://aws.amazon.com/batch); and how product managers, scientists, and engineers can work together to create customer-facing products based on these cloud-enabled algorithms.

## Step One: Containerize All The Algorithms

**_Imagine yourself an atmospheric scientist_**.  You have spent years diving deeply into your subject domain, poring through and adding to the scientific literature, performing countless experiments, and seeking objective truth through the scientific method.  While computer models are commonplace in the field of atmospheric research, you see them as a tool; it's the output of these models that is most interesting to you.  You are not interested in the minutiae of how an outdated ethernet cable may be bottlenecking the inter-timestep syncing of sub-domain boundary conditions between compute nodes in your company's linux cluster (just to give a realistic but sufficiently tedious sounding example).  You are not interested in the site licensing restrictions of your commercial FORTRAN compiler (to be fair, nobody is).  You are not interested in how to maintain 4 concurrent installations of a supporting software library and navigating symbolic links that point to symbolic links that point to symbolic links that hopefully do but probably do not point to what you need (it's OK, you can admit to reinstalling your entire operating system to get rid of your 7 different python installations to [just start over](https://xkcd.com/1987)).  You just want to run your computer models in an easy and reproducible way.  Let's have a look at how Docker can help alleviate these frustrations and aid in your research.

**_But first, what is Docker?_**  Docker is an open-source software tool that allows you to bundle up a piece of software along with all of its dependencies, from 3rd-party software libraries and supporting data files all the way down to the operating system directory structure (only the linux kernel itself is excluded).  This bundle of software is called a Docker image.  Docker images can easily be defined using a simple text file called a Dockerfile.  Once a Docker image is built, it can be executed, at which point it is called a Docker container.

**_And how does this help me?_**  A Docker container runs in isolation and is therefore not impacted by the rest of the host system (say goodbye to your problems with conflicting supporting libraries, environment variables, etc.).  Once created, a Docker image does not change, so it can be executed as a container as many times as you'd like and it will behave the exact same way each time (hello, [reproducible research](https://en.wikipedia.org/wiki/Reproducibility#Reproducible_research)!)  Another byproduct of containerization is portability; because Docker images bundle everything required to run your code, they can easily be run on many host systems and cloud platforms and even shared with colleagues.

**_Is it really that easy?_**  Let's have a look.  Let's say you have a [Python](https://www.python.org) algorithm in a file called `algorithm.py`.  This "algorithm" uses [NumPy](http://www.numpy.org) (a very commonly used python library for scientific algorithms) to create an identity matrix with a size specified by a user input argument.

**`algorithm.py`**
``` python
import sys
import numpy as np
identity_matrix = np.eye(int(sys.argv[1]))
print("Here is your identity matrix:")
print(identity_matrix)
```
Alongside that file, you would create a plain text file called `Dockerfile`, in which you would specify a few basic things: which starting Docker image you'd like to use, what code to add, what dependencies to install, and how you'd like to execute the code.  For our starting Docker image we'll grab the [official Python 3 image](https://hub.docker.com/_/python/) from [Docker Hub](https://hub.docker.com), the biggest public Docker registry.  On Docker Hub you'll find officially supported Docker images for all commonly used operating systems, software frameworks, and programming languages.   We'll then use Pip to install our required NumPy software library dependency and add our `algorithm.py` file to the root directory.  The `ENTRYPOINT` and `CMD` options specify how to execute our algorithm when the Docker container is run and what default command line argument to pass in to our identity matrix algorithm, respectively.

**`Dockerfile`**
``` docker
FROM python:3
RUN pip install numpy
COPY algorithm.py /
ENTRYPOINT [ "python", "/algorithm.py" ]
CMD [ "2" ]
```

Once Docker is [installed](https://docs.docker.com/install) on your system, we can build our Docker image and tag it with the name `algorithm` using the following command executed in the same directory as our `Dockerfile`:

```
docker build -t algorithm .
```

Docker will take some time initially to download the Python Docker image from Docker Hub and to install NumPy, but subsequent build commands would take less time as these items are cached.  Once the Docker image has been built, we can run our containerized algorithm with the Docker run command.  Let's provide an input argument value of 4 to create a 4x4 identity matrix.

```
docker run algorithm 4
```

Success!  We've run our algorithm in a Docker container and produced the ever-important 4x4 identity matrix as seen below in all its majesty.

```
Here is your identity matrix:
[[1. 0. 0. 0.]
 [0. 1. 0. 0.]
 [0. 0. 1. 0.]
 [0. 0. 0. 1.]]
```

With this newly minted Docker image, we could run our algorithm as many times as we want.  We could install other versions of Python or NumPy on our host machine without impacting the versions in our container.  We could share our `Dockerfile` and `algorithm.py` file with colleagues so that they could tweak our algorithm or run it as-is to see the same results.  We could upload our Docker image to a Docker image repository to make it available to the public, or, as we will see in the next section, to make it accessible to cloud-based operational systems.

This is, of course, a trivial example, but it is important to show to emphasize just how simple it is to get up and running with Docker.  Remember, you're an atmospheric scientist, and your primary focus is on developing and running algorithms in a simple, repeatable way.

## Step Two: Scale Up With AWS Batch

**_Imagine yourself a software engineer._**  You are tasked with working with your colleague (alter-ego?), the atmospheric scientist to not only help develop scientific algorithms, but also to enable these algorithms to be executed on the cloud.  You and your scientist colleagues often have projects that require scientific algorithms and models to be run hundreds or even thousands of times with different input parameters.  For example, an ensemble weather forecast is generated by running a weather forecast model many times, each with slightly different initial input conditions.  Due to the fundamentally chaotic nature of the equations that govern the Earth's atmosphere, slight perturbations to input parameters in weather models can actually lead to drastically different end results.  By running the model many times, an ensemble forecast can better show the range of possible weather forecast outcomes.  But running weather forecast models is computationally intensive and often take hours to execute even on supercomputers with hundreds or thousands of CPU cores available.  Running these models in a Docker container on your local laptop is not going to get the job done.

Traditionally, these weather forecast models are run on either a Linux compute cluster or supercomputer located at your company, university, or dedicated supercomputing facility.  While this method is certainly effective and has valid use cases, there are some downsides.  For one, getting access to these computing resources often requires proposal writing, money, and time.  Should you decide to obtain a supercomputer or compute cluster instead of accessing an external facility, purchasing and maintaining these systems is a serious commitment, and the only way they can remain cost-effective is to try to keep them running at full capacity 100% of the time for years on end.  To attempt to keep these machines running at full capacity, they are also generally shared with other users, so once you gain access you will likely have to put your compute jobs in a queue and wait your turn before they can be executed.  Supercomputers also tend to be rigid in terms of which compilers are available and which libraries can be installed.  It's often the case that you will need to spend significant time and resources figuring out how to get your code compiled and optimized to take advantage of the supercomputer's abilities.  While there are scenarios in which these downsides are worth the effort, they have a cumulative effect of preventing rapid iteration on new ideas.

Wouldn't it be nice if we could shoehorn our Docker containers into a high-performance computing environment without all those headaches?  Enter, AWS Batch.  AWS Batch is a fully managed service (i.e. you don't have to have a team of experts to set it up or keep it running) that lets you run your Docker containers on the cloud for your batch processing workflows (batch processing simply describes a computer program that does not require human interaction while it's executing; most scientific algorithms and models fit that description).  After some initial configuration, AWS Batch allows you to submit compute jobs to a job queue, and AWS Batch handles spinning up the required computational resources to execute the jobs.  After the jobs are done, the compute resources are terminated.  You only ever pay for the compute resources that your jobs actually need.  And because AWS itself houses an absolutely enormous number of underlying compute resources, you can execute hundreds, thousands, or even tens of thousands of compute jobs at the same time!  Getting started quickly, paying only for what you use, and being able to massively scale up the number of compute resources available to you allows scientists and engineers to not only accomplish their current workloads more easily, but also to ask new types of questions that could not be answered without this degree of agility and scalability.

While AWS Batch is simple to configure for somebody with AWS experience, it is likely outside the scope of what our scientist colleagues would ideally like to worry about.  Let's take a look at how we can build a more comprehensive cloud-native platform centered on AWS Batch for enabling scientific workflows that has a clear boundary of responsibilities between software engineers or systems administrators and scientific "component" developers (i.e. those who are responsible for packaging scientific algorithms into Docker containers).

![Scientific Component Developer Workflow](/assets/scientific-component-developer-workflow.png)

In the figure above, let's start at the bottom with the scientific component developer.  This developer can log in to an [AWS Workspace](https://aws.amazon.com/workspaces/), a cloud-hosted Linux or Windows desktop environment in which the developer can write algorithm code, test Docker containers at a small scale, examine output results, and access datasets stored in the company's "[data lake](https://aws.amazon.com/big-data/datalakes-and-analytics/what-is-a-data-lake/)".  A data lake is a centralized repository of a company's data, in this case stored on [AWS S3](https://aws.amazon.com/s3/) and fronted by an [AWS File Gateway](https://aws.amazon.com/storagegateway/file/) to enable access through a network file share.  Once a component developer is satisfied with an algorithm developed on their workspace and wants to try it out at scale on AWS Batch, the next step is for the component developer to push their algorithm code to a version control repository, in this case [AWS CodeCommit](https://aws.amazon.com/codecommit/).  At this point, the process is handled automatically by the continuous integration/continuous delivery platform that has been preconfigured by the system administrator.  [AWS CodePipeline](https://aws.amazon.com/codepipeline/) and [AWS CodeBuild](https://aws.amazon.com/codebuild/) are used to automatically build the Docker image based on the newly pushed code and publish it to an [AWS ECR Repository](https://docs.aws.amazon.com/AmazonECR/latest/userguide/Repositories.html).  From here, the Docker image is available to a preconfigured AWS Batch setup.  Once this automated process is complete, the component developer can log in to the [AWS Management Console](https://aws.amazon.com/console/) (a web-based graphical user interface for AWS) and submit compute jobs to AWS Batch to test out the new containerized algorithm.  AWS Batch is configured to be able to read from and write to the same data lake that the developer accesses through their Workspace, obviating the need to copy large datasets around between the AWS Batch processing system and the developer's Workspace.  If the initial algorithm testing goes well on AWS Batch, the component developer is also able to submit large numbers of jobs to AWS Batch, either through the Management Console or by scripting their job submissions using the [AWS CLI](https://aws.amazon.com/cli/) or one of many [AWS SDKs](https://aws.amazon.com/tools/) in the language of their choice.

With a container-centric, cloud-native scientific computing platform like this in place, there is no on-premise infrastructure to manage or provision, all compute and data resources are effectively infinitely scalable, and scientists and engineers are able to rapidly develop and deploy containerized algorithms at scale.

## Step Three: Automate and Productize on AWS

**_Imagine yourself a product manager._**  You are tasked with leveraging the scientific research and algorithm development of your company's scientists and engineers to define and build products and capabilities that service your customers' requirements and expectations.  You have found through years of experience that the most important factor for your team's success is *agility*.  Customers don't always know exactly what they need before you show them the art of the possible.  You need to be able to rapidly iterate on customer feedback to create prototype demonstrations of capability to ensure that you are building what your customers actually need.  Let's have a look at how we can build on the foundation that our colleagues the scientist and the software engineer have created in the previous sections to productize their algorithms and build scalable, stable, and scientifically sound user-facing capabilities.

![Productizing on AWS](/assets/productizing.png)

In the figure above, let's start on the right side with the scientific component development process.  Recall that our scientists and engineers have already established a cloud native framework for developing and running containerized algorithms using AWS Batch and other AWS services.  With the approach we describe here, the Docker images created by our scientists and engineers can be used as-is in our operational system.  There is generally no compelling reason to rewrite our scientific algorithms "for production". Maintaining a single code base between scientists, engineers, systems administrators, security specialists, and product managers ensures that everyone is on the same page with respect to scientific algorithm integrity, stability, performance, security, and general code "correctness".  Getting back to the figure itself, the output from the scientific component development process can be considered to be the containerized algorithms sitting on a Docker image repository, in this case an AWS ECR Image Repository.  And just as in the previous section, the Docker images from this image repository can be used in an AWS Batch environment that mimics (or is equivalent to) the Batch environment from the scientific component development framework.  What changes is the method by which Batch jobs are submitted.  For scientists and engineers, Batch jobs were submitted either in an ad hoc fashion for testing or possibly by scripting a series of job submissions using the AWS API or an AWS SDK.  When it comes to creating a product for a customer to use, the goal is to abstract away the underlying AWS infrastructure and to provide a clean and simple interface tailored to customer use cases.  To this end, Batch jobs are submitted by [AWS Lambda](https://aws.amazon.com/lambda/) functions which are tied to an [AWS API Gateway](https://aws.amazon.com/api-gateway/).  It's this API Gateway that is providing the customer-facing interface to our capabilities.  The API contains a series of operations that can be performed, e.g. `submit job`, `list jobs`, `cancel job`, `list results`, `get result`, etc.  Requests to the API are tied to any number of underlying AWS services.  Requests such as `submit job` are tied to a Lambda function that submits the job to an AWS Batch Job Queue, whereas requests like get result are tied directly to files on S3.  For each of these API requests, users are authenticated to ensure they have access to the API using [AWS Cognito](https://aws.amazon.com/cognito/).  While some customers will want to interact directly with the API to enable automated machine-to-machine communication with our capabilities, other customer use cases will call for a graphical user interface.  There are many ways to accomplish this, including mobile and desktop applications, but the figure above shows how a static web-based user interface (WebUI) can be hosted on AWS S3 while still using the same API behind the scenes.

In addition to allowing customers to access individual scientific algorithms, our system shown above can also be configured to perform more complex operations involving multiple scientific algorithms run sequentially or in parallel.  For example, AER's Air Quality group uses this architecture in a customer-facing product called **AQcast** to allow customers to run a series of air quality related models to produce results that can be used for government regulation compliance purposes.  In order to perform this particular comprehensive analysis, many individual scientific algorithms and models must be orchestrated: the Weather Research and Forecasting Model ([WRF](https://www.mmm.ucar.edu/weather-research-and-forecasting-model)) must be run to downscale weather for each day in the analysis period; the [EPA Emissions Modeling Platform](https://www.epa.gov/air-emissions-modeling/emissions-modeling-platforms) must be run (which itself is comprised of dozens of individual models); and the Community Multiscale Air Quality Modeling System ([CMAQ](https://www.epa.gov/cmaq)) must be run against the output of the previous steps.  All total, a typical analysis will typically involve hundreds or thousands of individual Batch jobs, some requiring up to 72 CPU cores each, and the analysis can take many hours to perform.  Each model in this system requires scientific expertise to understand and technical expertise to compile and get working, and input and output datasets can measure in the terabyte range between each processing step, so customers can save an immense amount of time and effort by leveraging AQcast's model-as-a-service approach.

It's only by bringing together AER's scientists, engineers, and products managers with an architecture and workflow shown in this section that complex systems like AQcast can be built.  These systems deliver on the product manager's priority of agility.  New algorithms and models can easily be added to these systems and chained together with other algorithms to create higher-order capabilities.  New capabilities can be added to the API and linked up to the required backend services.  These systems can be created from scratch in minutes using [AWS CloudFormation](https://aws.amazon.com/cloudformation/) templates and cost next to nothing until the actual models are executed.  There is no hardware to purchase.  There is nobody else to compete with for processing time.  The number of jobs in the queue can scale from zero to one million.  With so many pain points removed compared to traditional on-premise infrastructure, the product manager is free to quickly iterate through concepts with the customer to align with their requirements.

## Takeaways

We've shown how scientists, engineers, and product managers can work closely together, armed with Docker containers, AWS services, and a DevOps mentality to go from research to operations.  Here are some takeaways:
* Docker containers help to encapsulate complexity, enabling portable and reproducible algorithms that can be run locally or on AWS.
* By using the same containerized algorithms all the way from scientific research to production, the integrity of the scientific algorithms is more easily maintained.
* AWS managed services give scientists and engineers leverage to accomplish more with less "undifferentiated heavy lifting".  By focusing on core business competencies and outsourcing the rest to AWS services, small teams can do big things while remaining agile.
* The on-demand and scalable nature of AWS lets scientists, engineers, and product managers ask new types of questions, unbounded by traditional limitations imposed by on-premise compute and data resources.