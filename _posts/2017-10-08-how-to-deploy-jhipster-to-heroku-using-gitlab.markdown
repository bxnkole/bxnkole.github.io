---
title:  "How to deploy JHipster to Heroku using GitLab"
description: How to deploy JHipster to Heroku using GitLab Continuous Integration/Continuous Delivery (CI/CD)
category: java, angular, spring-boot, jhipster, gitlab, heroku
---

I was trying to deploy my newly generated JHipster application to Heroku and ran into a few a dark alleys. I'm out, thanks to
the very nice CI/CD integration which is available on GitLab.

To kick off this tutorial, lets start by scaffolding a simple JHipster application by running 'yo jhipster' from the terminal. 
Once the scaffolding is complete, create a **.gitlab-ci.yml** file in the root of the project. This file is the source of truth for GitLab's CI/CD 
integration. Full reference/documentation for this file is available at [https://docs.gitlab.com/ee/ci/yaml/](https://docs.gitlab.com/ee/ci/yaml/)

On the Heroku side of things, we need to setup a project for our sample application. Let call the application 'cool-heroku-app'.
Then we need to obtain an API key (user-scoped, not project-scoped) from user settings. This API key will be used to construct a url later.

![API Key From Heroku](/assets/images/heroku-gitlab-ci-post/4.png)

So, back to our code. Inside the **.gitlab-ci.yml** we created earlier, we add a job for the Gitlab runner to execute when a push is triggered. 
Let's call it 'push_to_heroku'. Two scripts are going to be executed in this job. The first will perform a checkout from the branch that is
receiving the push. Let's call the new branch 'heroku-bed'. The role of the second script is to execute a 'git push' command to Heroku. 
Since we don't have access to an interactive console at this point, we cannot do 'heroku login'.
Therefore, we need to specify connection parameters as part of the Heroku url which we intend to push to. 
There is also an 'only' section in our 'push-to-heroku' job. We use this section to specify which branches we are listening on for push events in 
order to activate this job. 

The full **.gitlab-ci.yml** file is displayed below.

{% highlight yml %}
push_to_heroku:
  script:
    - git checkout -b heroku-bed
    - git push https://heroku:$API_KEY@git.heroku.com/cool-heroku-app.git heroku-bed:master --force
  only:
    - master
{% endhighlight %}

I have added the API_KEY as a secret variable in GitLab, which is why I am using the dollar-sign notation. Its more secure that way.
You can choose to add it directly to this file if you're just playing around.

Next, we create a GitLab repository to push our application to. GitLab *automagically* detects the presence of the **.gitlab-ci.yml** file
in the root of our project. Thankfully, GitLab has a number of default runners which will pick up any pre-defined jobs and execute 
appropriately. So, after pushing to GitLab, we navigate to the Jobs page of our project (under CI/CD). Here, we see that the 'push_to_heroku' job 
should be running already. If all goes well, the pipeline will succeed and the build will pass.

![Pipeline Status](/assets/images/heroku-gitlab-ci-post/3.png)

Questions? Let me know in the comments.