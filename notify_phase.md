**_Before beginning this step, be sure to be at a command line prompt from your prepared working environment.  This will either be your local machine, or within the provided container._**

#### Reminder: Working in the docker container

```
# Start a clean instance of the container
docker run -it --name cicdlab hpreston/devbox:cicdlab

[root@cf95a414877e coding]# exit

# If you need to restart an exited container
# Verify that you have  a container in a stopped state
docker ps -a

CONTAINER ID        IMAGE                         COMMAND             CREATED             STATUS                        PORTS               NAMES
cf95a414877e        hpreston/devbox:cicdlab       "/bin/bash"         2 minutes ago       Exited (0) 10 seconds ago                         cicdlab

# Restart your stopped container
docker start -i cicdlab

[root@cf95a414877e coding]#
```

[item]: # (slide)

## Monitor and Notify Phase

[item]: # (/slide)

In this step, we will configure the job to send a notification to the Spark room indicating whether the build was successful or failed.

[item]: # (slide)

## Update the .drone.yml configuration

[item]: # (/slide)

In the root of the code repository is a file _.drone.yml_ that provides the instructions to drone on what actions to take upon each run.  We will be updating this file at each stage of the lab to move from Continuous Integration -> Delivery -> Deployment.

**_In this step you will be entering several commands in a terminal window.  These need to be run from your local repo directory.  If you followed the directions when cloning the repo locally, this command will place you in the correct directory_**

```
cd ~/coding/cicd_demoapp
```

### Steps
1. Open the _.drone.yml_ file in your editor.  We will be adding the next phase, _notify_ to the existing configuration.  Update your drone file to look like this.
    1. **NOTE: You can simply copy and paste the contents from below directly into your file.  The notation of _$$VARIABLE_ references details stored encrypted within the .drone.sec file.  Do NOT replace them with clear text credentials.**

[item]: # (slide)

```
build:
    build_starting:
        image: python:2
        commands:
            - echo "Beginning new build"
    run_tests:
        image: python:2-alpine
        commands:
            - pip install -r requirements.txt
            - python testing.py
	
publish:
    docker:
        repo: $$DOCKER_USERNAME/cicd_demoapp
        tag: latest
        username: $$DOCKER_USERNAME
        password: $$DOCKER_PASSWORD
        email: $$DOCKER_EMAIL
	
deploy:
    webhook:
        image: plugins/drone-webhook
        skip_verify: true
        method: POST
        auth:
            username: $$MANTL_USERNAME
            password: $$MANTL_PASSWORD
        urls:
            - https://$$MANTL_CONTROL/marathon/v2/apps/class/$$DOCKER_USERNAME/restart?force=true

notify:
    spark:
        image: hpreston/drone-spark
        auth_token: $$SPARK_TOKEN
        roomId: $$SPARK_ROOM            
```

[item]: # (/slide)

2. As part of the security of drone, every change to the _.drone.yml_ file requires the secrets file to be recreated.  Since we've updated this file, we need to re-secure our secrets file.

[item]: # (slide)

    ```
    # Replace USERNAME with your GitHub username
    drone secure --repo USERNAME/cicd_demoapp --in drone_secrets.yml
    ```

[item]: # (/slide)

    1. If the command above gives the error `Error: you must provide the Drone server address.`, you likely closed the terminal window after the Environment Prep step.  Run these commands to reset the environment variables.

        ```
        export DRONE_SERVER=http://DRONE_SERVER
        export DRONE_TOKEN=<your token>
        ```

3. Now commit and push the changes to the drone configuraiton and secrets file to GitHub.
	* **Remember if you are using an IDE, be careful not to commit & push the changes to the file drone_secrets.yml**

[item]: # (slide)

    ```
    # add the file to the git repo
    git add .drone.sec
    git add .drone.yml

    # commit the change
    git commit -m "Added Notify Phase"

    # push changes to GitHub
    git push
    ```
    
[item]: # (/slide)

4. Now check the Drone web interface. A new build should have kicked off.  Also, watch for the Spark message to come through in the client. 

[item]: # (slide)

    ![Drone Build](images/drone_5th_build.png)

[item]: # (/slide)

5. If the build reports a **Failure**, check the log to see what might have gone wrong.  Common reasons include forgetting to re-create the secrets file, forgetting to commit the secrets file after recreating, or mis-entered credentials in the plain text secrets file.
6. At the end of the build check the Spark Room.  You should see a status notification regarding the state of the build.

[item]: # (slide)

    ![Spark Notification](images/spark_notify1.png)

[item]: # (/slide)


[item]: # (slide)

## Current Build Pipeline Status

![Final Diagram](images/stage_final_diagram.png)

[item]: # (/slide)

Okay, so drone said it did something and we got a Spark message... but you may be wondering what actually happened.  This image and walkthrough shows the steps that are occuring along the way.

1. You committed and pushed code to GitHub.com
2. GitHub sent a WebHook to the Drone server notifying it of the committed code.
3. Drone checks the _.drone.yml_ file and executes the commands in the _build_ phase. During this phase, Drone has two steps: 
	* *build_starting* 	
  		* Fetch a container from hub.docker.com to begin the build.  This container is identified in the `image: python:2` line of the drone config file.  Drone will run the commands listed in this section
	* *run_tests*
  		* Fetch a container from hub.docker.com to do the testing.  This container is identified in the `image: python:2-alpine` line of the drone config file.  Drone will run the tests described by the commands listed in this section

4. Drone checks the _.drone.yml_ file and executes the commands in the _publish_ phase. During this phase, Drone will: 
  * Build a Docker Container using the Dockerfile definition included in the Git repo
  * Push the container up to hub.docker.com using the credentials contained in the secrets file
5. Drone checks the _.drone.yml_ file and executes the commands in the _deploy_ phase. In this phase, the following actions will take place: 
  * Drone sends a WebHook command to Marathon to cause an application restart
  * Marathon pulls the new container from hub.docker.com containing the code changes
6. Drone checks the _.drone.yml_ file and executes the commands in the _notify_ phase. During notification, Drone will check the staus of the build and then send notifications to the Spark room.
  * If the build was successful, send a Success notification
  * If the build failed, send a Failure notificiation and blame someone.

[item]: # (slide)

## Bonus Step!

[Bonus - CICD In Action!](bonus.md)

[item]: # (/slide)

You now have a full CICD pipeline setup, but we haven't actualy made any updates to the application.  Move onto the next step to become a DevOps enabled developer and make an application change and watch it go to production!
