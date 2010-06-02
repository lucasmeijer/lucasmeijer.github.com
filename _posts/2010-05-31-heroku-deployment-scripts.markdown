Why?
----

I love heroku. Their elegant 'git push' deployment process is an effective way to deploy code. However, for all it's elegance, things start to break down once you want to do some more advanced things that capistrano easily took care of. 

1. First, I want to deploy to any remote environment with a single task, such as:
`rake deploy:production` or `rake deploy:staging`

2. I needed a way to package and upload assets to a CDN on each deploy without relying on some git post-commit hook hackery.

3. I want to deploy the current working branch to the remote environment. This will let me seamlessly create a local branch, make some changes, then deploy to staging without having to alter any git configs or muck around with refs.

4. I want to ensure that only master gets pushed to production. Deploying a local working branch to production could be trouble. 

The solution? Whip up a Rake task that wraps these requirements in a single command:

`rake deploy:production`

The Deploy Task
-------------

First, lets define our environments and their cooresponding git remotes:

  ENVIRONMENTS = {
    :staging => 'myapp-staging',
    :production => 'myapp-production'
  }

The ENVIRONMENTS hash is used to dynamically build a deploy task for each environment:

  $ > rake -T deploy
  rake deploy:production
  rake deploy:staging

The deploy task itself is fairly straight forward. First we will get the name of the current local working branch. This is used to push the current branch to Heroku without first merging into master. Once we know the branch name, we can pass it, along with the environemnt, to three tasks: before_deploy, update_code, and after_deploy. Here are the Rake tasks:

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
      # ...
    end
  
    task :after_deploy, :env, :branch do |t, args|
      # ...
    end
  
    task :update_code, :env, :branch do |t, args|
      # ...
    end
  end

Update Code
-----------
The update_code task makes the `git push` call to Heroku. It's smart enough to push your current local branch to Heroku even if even if the update is not a fast-forward. This is usually the case if you use GitHub as your primary code repo and push many branches to staging without merging first.
  
  namespace :deploy do
    task :update_code, :env, :branch do |t, args|
      FileUtils.cd RAILS_ROOT do
        `git push #{ENVIRONMENTS[args[:env]]} +#{args[:branch]}:master`
      end
    end
  end

The git command would look something like `git push myapp-staging +bugs:master`. This tells git to push your local bugs branch to your myapp-staging remote by updating master on Heroku.

Before Deploy
-------------
The before_deploy task is a hook where you can add custom tasks to be ran before your code is pushed to Heroku. For example, you may have an 'assets:deploy' task which bundles and deploys assets to S3:

  namespace :deploy do
    task :before_deploy, :env, :branch do |t, args|
      Rake::Task['assets:deploy'].invoke(args[:env], args[:branch])
    end
  end

You can create as many deploy:before_deploy tasks as you need. As long as they are declared AFTER this deploy task, Rake will pick them up and run them in order. See XXX for more info on how Rake has an alias_task_chain of sorts.

After Deploy
------------
Similar to before_deploy, after_deploy is a hook where you can add custom tasks to be ran after your code is pushed to Heroku. You can have as many of these as you need too. A common task is notifying Hoptoad after you've successfully deployed:

  namespace :deploy do
    task :after_deploy, :env, :branch do |t, args|
      Rake::Task['hoptoad:deploy'].invoke
    end
  end

Non-master deploy prompt
------------------------

Pushing branches willy-nilly to production could be trouble. As a result, the before_deploy task will prompt you when deploying a non-master branch to production:

  task :before_deploy, :env, :branch do |t, args|
    # Ensure the user wants to deploy a non-master branch to production
    if args[:env] == :production && args[:branch] != 'master'
      print "Are you sure you want to deploy '#{args[:branch]}' to production? (y/n) " and STDOUT.flush
      char = $stdin.getc
      if char != ?y && char != ?Y
       puts "Deploy aborted"
       exit 
      end
    end
  end

This code could be simplified by using the Highline gem, but I didn't want to add another development dependency for this one spot. Feel free to improve that for yourself.

And with that, we have a deployment task that holds up to Heroku's simplicity. I've used this in projects both large and small with great success. Enjoy.


Notes:
  * Migrations are not automatic since they could be big or small and typically involve maintenance mode. Plus I don't use AR
  * These tasks use awk and grep which I am pretty sure aren't on Windows, so it likely won't work there.
  * Extend with task overriding



------
ref 1 http://jqr.github.com/2009/04/25/deploying-multiple-environments-on-heroku