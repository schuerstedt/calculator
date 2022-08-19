CI Pipeline independence by using docker
========================================

Introduction
============

During our last DevOps class, I was asked to run a Hackathon style session. Ok, it was mostly me and the participants watching my trial and error to do a proof of concept for the following idea:

Can I use a docker build environment to get a pipeline independence?

Background: we are playing mostly with Azure Pipelines and Github Action. I add a Circle pipeline mostly to show, that each tool has a similar, but own syntax.

As I always start the building of a pipeline with the slogan: own (and know) your build process, I start with my local build environment and then move that to the pipeline environment.

So all the same, right?

Even when just playing around, the answer is NO. All different, some easier, some have samples working out of the box etc.

The idea came up, as most of the pipeline environments can run docker, why not use docker to run the build inside the run environment. If we do that self-sustaining, that would be even better. Just one docker run and you get your result.

As a sample we use a github sample from Microsoft Az-400 devops training labs: [Integrating External Source Control with Azure Pipelines](https://github.com/MicrosoftLearning/AZ400-DesigningandImplementingMicrosoftDevOpsSolutions/blob/master/Instructions/Labs/AZ400_M03_L06_Integrating_External_Source_Control_with_Azure_Pipelines.md).

Here the tasks are to setup a Azure Pipeline by using Github as repository (caution: if one want to build that sample you need to have a pipeline set up in your billing. The free tier is available on request only. So do not expect to do and run the sample immediately).

The documented build for this node application is:

npm install

npm test

where as npm test runs a Mocha based test to do a unit test.

Get a docker to run the build
-----------------------------

Going to docker hub, we found the node docker package: <https://hub.docker.com/_/node>. It is an official docker image, so should be safe to use (always check the source when using open source 😉).

In the documentation is a sample to tun a single node.js script:

*$ docker run -it --rm --name my-running-script -v "$PWD":/usr/src/app -w /usr/src/app node:8 node your-daemon-or-script.js*

We started from that and opened the node in interactive mode to see, what was running in the container:

*$ docker run -it --rm node sh*

We played around a little to see if npm is working etc.

Next step was to map our git repository into the container:

*$ docker run -it --rm --name my-running-script -v "$PWD":/usr/src/app node sh*

That way we could figure our which directories to use and to run the npm install and npm test command.

Once we were familiar with that, we decided to use a script to run those commands which let to

build.sh

npm install

HOME=$PWD

npm test

The npm test keeps failing with an access error message. After a little googling we found out the used nyc script inside npm test has a hard coded $HOME in it. That why we set the HOME directory before calling npm test.

As npm test is last comment in build.sh the exit code from it propagates up to the docker run. If npm test runs well, it exits with 0, if it fails with >1\. That was important, as the CI actions in the pipeline needed to get this exit code to decide, if the CI build was successful or not.

Eventually we succeeded with the following run command in the pipeline:

*docker run -it --rm --name my-running-script -v "$PWD":/usr/src/app node sh /usr/src/app/build.sh*

Cool. Our first solution was running well and we could use the docker run in Azure Pipeline as well as in Guthub Actions.

One thing we did not like, was that the repository came from the working directory in the pipeline virtual machine. That worked fine in Azure and Github, but if someone wanted to test the CI docker command locally, one had to clone the repo first.

We were up to a quest to do all that inside of the docker container...

Finally -- a docker container build getting its own command from our repository
------------------------------------------------------------------------------

We liked the idea to have the build script in our repository as well as the pipeline code. One source for everything.

Now we were looking to clone the repository inside of the container and run the build from there.

The solution was quite simple by using bash -c with a list of commands:

*docker run --rm node /bin/bash -c "cd usr/src; git clone https://github.com/schuerstedt/calculator; cd calculator; sh ./build.sh"*

First we validated, if the node container had git installed. It did 😊.

Our final one line command was:

cd /usr/src # go to the working directory

git clone...  # clone our repository

cd calculator  # go into the cloned directory

sh ./build.sh # can call our build.sh script

Voila, that was working.

Now we had a docker run which was completely independent of the run time environment. Only prerequisite was docker.

We tested that with Azure Pipeline and Github Actions and it worked quite well.

Go ahead and test it on your own. Get your hands dirty. Enjoy.