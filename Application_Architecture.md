# Container sample application

The Steps
Here’s an overview of the steps outlined below:

* A – Pre-Requisites
* B – Clone The 3 app Repositories
* C – Create The Infrastructure
* D – Set Up Secrets And Keys
    * D1 – Container Registry Login Secret Setup
    * D2 – Set Up Repository Access Token
    * D3 – Install And Set Up FluxCD
* E – Build And Run Your Application

## A – Pre-Requisites
You will need:

* A Gitlab GIT Repo account
* Access to the registry (Nexus / Azure container registry)
* Application container built in registry
* "Insert UBS pre reqs here"
* could be a possible branch here for dev work and app work. 

## B – Clone the Repositories
Clone these three repositories to your own Gitlab GIT Repo account:

* https://GitlabGITRepo/"projectname"/gitops-example-app
* https://GitlabGITRepo/"projectname"/gitops-example-deploy
* https://GitlabGITRepo/"projectname"/gitops-example-infra (?? If this just a call to something else - ARM / TF template)

## C – Create the Infrastructure
TBD - Work out the best way of deploying what they gave already built (Gitlab pipeline / TF module etc)

## D – Set Up Secrets And Keys
TBD - Work out the best way of deploying what they gave already built (Gitlab pipeline / TF module etc)


## E – Build And Run Your Application
To deploy your application, all you need to do is make a change to the application in your gitops-example-app repository.


An overview of the flow of this
Step 1a, 2 and 3
Go to:

https://Gitlab GIT Repo.com/<YOUR Gitlab GIT Repo USERNAME>/gitops-example-app/blob/main/Dockerfile

and edit the file, changing the contents of the echo command to whatever you like, and commit the change, pushing to the repository.

This push (step 1a above) triggers the Docker login, build and push via a Gitlab GIT Repo action (steps 2 and 3), which are specified in code here:

https://Gitlab GIT Repo.com/ianmiell/gitops-example-app/blob/main/.Gitlab GIT Repo/workflows/main.yaml#L13-L24

This action uses a couple of docker actions (docker/login-action and docker/push-action) to commit and push the new image with a tag of the Gitlab GIT Repo SHA value of the commit. The SHA value is given to you as a variable by Gitlab GIT Repo Actions (Gitlab GIT Repo.sha) within the action’s run. You also use the DOCKER secrets set up earlier. Here’s a snippet:

    - name: Log in to Docker Hub
      uses: docker/login-action@f054a8b539a109f9f41c372932f1ae047eff08c9
      with:
        username: ${{secrets.DOCKER_USER}}
        password: ${{secrets.DOCKER_PASSWORD}}
    - name: Build and push Docker image
      id: docker_build
      uses: docker/build-push-action@ad44023a93711e3deb337508980b4b5e9bcdc5dc
      with:
        context: .
        push: true
        tags: ${{secrets.DOCKER_USER}}/gitops-example-app:${{ Gitlab GIT Repo.sha }}
Step 4
Once the image is pushed to the Docker repository, another action is called which triggers another action that updates the gitops-example-deploy Git repository (step 4 above)

    - name: Repository Dispatch
      uses: peter-evans/repository-dispatch@v1
      with:
        token: ${{ secrets.EXAMPLE_GITOPS_DEPLOY_TRIGGER }}
        repository: <YOUR Gitlab GIT Repo USERNAME>/gitops-example-deploy
        event-type: gitops-example-app-trigger
        client-payload: '{"ref": "${{ Gitlab GIT Repo.ref }}", "sha": "${{ Gitlab GIT Repo.sha }}"}'
It uses the Personal Access Token secrets EXAMPLE_GITOPS_DEPLOY_TRIGGER created earlier to give the action the rights to update the repository specified. It also passes in an event-type value (gitops-example-app-trigger) so that the action on the other repository knows what to do. Finally, it passes in a client-payload, which contains two variables: the Gitlab GIT Repo.ref and the Gitlab GIT Repo.sha variables made available to us by the Gitlab GIT Repo Action.

This configuration passes all the information needed by the action specified in the gitops-example-deploy repository to update its deployment configuration.

The other side of step 4 is the ‘receiving’ Gitlab GIT Repo Action code here:

https://Gitlab GIT Repo.com/ianmiell/gitops-example-deploy/blob/main/.Gitlab GIT Repo/workflows/main.yaml

Among the first lines are these:

on:
  repository_dispatch:
    types: gitops-example-app-trigger
Which tell the action that it should be run only on a repository dispatch, when the event type is called gitops-example-app-trigger. Since this is what we did on the push to the gitops-example-app action above, this should be the action that’s triggered on this gitops-example-deploy repository.

The first thing this action does is check out and update the code:

      - name: Check Out The Repository
        uses: actions/checkout@v2
      - name: Update Version In Checked-Out Code
        if: ${{ Gitlab GIT Repo.event.client_payload.sha }}
        run: |
          sed -i "s@\(.*image:\).*@\1 docker.io/${{secrets.DOCKER_USER}}/gitops-example-app:${{ Gitlab GIT Repo.event.client_payload.sha }}@" ${Gitlab GIT Repo_WORKSPACE}/workloads/webserver.yaml
If a sha value was passed in with the client payload part of the Gitlab GIT Repo event, then a sed is performed, which updates the deployment code. The workloads/webserver.yaml Kubernetes specification code is updated by the sed command to reflect the new tag of the Docker image we built and pushed.

Once the code has been updated within the action, you commit and push using the stefanzweifel/git-auto-commit-action action:

      - name: Commit The New Image Reference
        uses: stefanzweifel/git-auto-commit-action@v4
        if: ${{ Gitlab GIT Repo.event.client_payload.sha }}
        with:
          commit_message: Deploy new image ${{ Gitlab GIT Repo.event.client_payload.sha }}
          branch: main
          commit_options: '--no-verify --signoff'
          repository: .
          commit_user_name: Example GitOps Bot
          commit_user_email: <AN EMAIL ADDRESS FOR THE COMMIT>
          commit_author: <A NAME FOR THE COMMITTER> << AN EMAIL ADDRESS FOR THE COMMITTER>>
Step 5
Now the deployment configuration has been updated, we now wait for FluxCD to notice the change in the Kubernetes deployment configuration. After a few minutes, the Flux controller will notice that the main branch of the gitops-example-deploy repository has changed, and try the apply the yaml configuration in that repository to the Kubernetes cluster. This will update the workload.

If you port-forward to the application’s service, and hit it using curl or your browser, you should see that the application’s output has changed to whatever you committed above.

And then you’re done! You’ve created an end-to-end GitOps continuous delivery pipeline from code to cluster that requires no intervention other than a code change!