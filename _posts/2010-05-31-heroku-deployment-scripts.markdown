---
layout: post
title: Heroku Deploy Script
---

I love [Heroku](http://heroku.com). Their elegant `git push` deployment is a great way to deploy code, but that's about it. If you've got extra work to perform, such as packaging and uploading assets to a CDN, you're out of luck.

The solution? Roll your own Rake task that sandwiches `git push` with hooks for additional tasks:

{% highlight bash %}
  rake deploy:production
{% endhighlight %}

## The Code
Throw this into `lib/tasks/_heroku/deploy.rake`. I'll explain the `_heroku` directory shortly.

{% highlight ruby %}
  # List of environments and their heroku git remotes
  ENVIRONMENTS = {
    :staging => 'myapp-staging',
    :production => 'myapp-production'
  }

  namespace :deploy do
    ENVIRONMENTS.keys.each do |env|
      desc "Deploy to #{env}"
      task env do
        current_branch = `git branch | grep ^* | awk '{ print $2 }'`.strip

        Rake::Task['deploy:before_deploy'].invoke(env, current_branch)
        Rake::Task['deploy:update_code'].invoke(env, current_branch)
        Rake::Task['deploy:after_deploy'].invoke(env, current_branch)
      end
    end

    task :before_deploy, :env, :branch do |t, args|
      puts "Deploying #{args[:branch]} to #{args[:env]}"
    end

    task :after_deploy, :env, :branch do |t, args|
      puts "Deployment Complete"
    end

    task :update_code, :env, :branch do |t, args|
      FileUtils.cd Rails.root do
        puts "Updating #{ENVIRONMENTS[args[:env]]} with branch #{args[:branch]}"
        `git push #{ENVIRONMENTS[args[:env]]} +#{args[:branch]}:master`
      end
    end
  end
{% endhighlight %}

## Why is this awesome?
There are a couple of interesting things this code does. Such as...

### Multiple Environments
The `ENVIRONMENTS` hash is used to dynamically build a deploy task for each environment:

{% highlight bash %}
  $ rake -T deploy
  rake deploy:production  # Deploy to production
  rake deploy:staging     # Deploy to staging
{% endhighlight %}

Just update the code so your environment names correspond to the proper git remote on Heroku. If you need more info on setting this up, check out Elijah Miller's [awesome writeup](http://jqr.github.com/2009/04/25/deploying-multiple-environments-on-heroku).


### Dynamic Branch Deployment

Your current local branch, whatever that may be, will be pushed to Heroku. This is awesome because a branch, say "bugs", can be deployed without first merging into master. It's smart enough to overwrite whatever is currently on Heroku even if the push is [not a fast-forward](http://rip747.wordpress.com/2009/04/20/git-push-rejected-non-fast-forward/). This is desirable if you use GitHub as your primary code repo and push many branches to staging without merging first.

### Hooks
Three extension hooks are available: **before_deploy**, **update_code**, and **after_deploy**. Because Rake is awesome, tasks with the same name will run in the order they were discovered. Thus, any task in your project with a name `deploy:before_deploy` will get executed in sequence as long as the task is defined **after** our primary deploy task. That's why the deploy task in the `_heroku` folder -- it would be loaded before our other rake tasks.

### Example Hooks

Here is a task I use to package assets then deploy to S3:
{% highlight ruby %}
  namespace :deploy do
    task :before_deploy, :env, :branch do |t, args|
      Rake::Task['assets:deploy'].invoke(args[:env], args[:branch])
    end
  end
{% endhighlight %}

Or how about this one that notifies Hoptoad after you've successfully deployed:
{% highlight ruby %}
  namespace :deploy do
    task :after_deploy, :env, :branch do |t, args|
      Rake::Task['hoptoad:deploy'].invoke
    end
  end
{% endhighlight %}

Pushing branches willy-nilly to production could be trouble. This task will prompt you when deploying a non-master branch to production:

{% highlight ruby %}
  task :before_deploy, :env, :branch do |t, args|
    # Ensure the user wants to deploy a non-master branch to production
    if args[:env] == :production && args[:branch] != 'master'
      print "Continue deploying '#{args[:branch]}' to production? (y/n) " and STDOUT.flush
      char = $stdin.getc
      if char != ?y && char != ?Y
       puts "Deploy aborted"
       exit 
      end
    end
  end
{% endhighlight %}

## Conclusion
And with that, we have a deployment process that combines Heroku's simplicity with capistrano's extensibility. Here is a gist of the full code: GIST HERE

Although this code has served me well in projects both large and small, I am excited to see what you can do with it! Enjoy.
