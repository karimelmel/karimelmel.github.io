---
layout: post
title:  "CloudFormer review part I - The stack"
date:   2020-09-25
---

Recently I was tasked to have a closer look at CloudFormer, a tool created by Amazon Web Services that helps create CloudFormation templates of existing resources within an account. At first glance I thought this was a completely new service, since it is still marked as Beta in the docs here: [https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/cfn-using-cloudformer.html](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/cfn-using-cloudformer.html), but the pictures in the documentation has the old GUI and there is also an old blog post mentioning [CloudFormer](https://aws.amazon.com/blogs/devops/building-aws-cloudformation-templates-using-cloudformer). 

To perform my assessment I start looking into the service, which is deployed through a CloudFormation stack. Getting a hold of the stack is my first priority ![](/image/cloudformerstack.jpg) and is very straight forward. By pretending to launch the stack I can view it in Designer, export it and start reviewing. If you want to have a look at the full stack I've uploaded it here [CloudFormer stack](https://gist.github.com/karimelmel/f5a0e9c975bc9b43fd3371c27662f090).

## The stack
The first element that of interest in the stack is the AMI. It references the an AMI per region, which is consistent to a very outdated version of Amazon Linux (not 2). For us-east-1 the AMI is ami-7f6aa912, which was last updated June, 22 2016. 
![](/image/ami.JPG)

Between then and now there is a total of 591 security advisories issued and more than 1000 CVEs. On top of that, the server is externally exposed so this can become interesting. The complete list can be found here (https://alas.aws.amazon.com/)

Taking a closer look at the instance profile it has access to a multitude of API calls, mainly **Get**, **Describe**, and **List** actions. The most powerful action would be s3:Get* without any resource constraint, meaning it can be used to read items from S3 buckets with weak configuration, such as allowing ${AccountId}:root which I commonly come across.

As for the network it either uses the Default VPC in an account *or* it creates a new VPC called CloudFormerVPC with an Internet Gateway and public route attached to it.

### Bootstrap
The bootstrap in the script performs multiple actions, including:
- Installing and configuring all dependencies
- Installing CloudFormer
- Generating a self-signed certificate
- Configuring the web service with the password provided in the template
- Starting the web service


### The web service
Once the stack is launched, the server is exposing a Basic Authentication endpoint over https through its public interface 

![](/image/auth.JPG)

The username/password provided in the stack will give you access the the interface through SSH. From here the application is exposed. 

What is interesting here is if you break into the web service in a production account you can perform reconnaissance by discovering what AWS resources is available in the environment and have it printed out in CloudFormation. Since it does not read existing cfn templates, I did not find a way to directly extract secrets.
![](/image/recon.JPG)

## Accessing the instance
What I am interested in to better understand what is going on, is getting access to the instance so I can look at the source code directly. Backdooring the instance is quite trivial, you can modify the cloudformation stack to include an SSH key or modify the embedded userdata before deploying the stack. 
I simple added a key that I have access to in the cfn stack and attached a security group allowing port 22 from my public IP. 
![](/image/cfnupdate.JPG)

This gives me acccess to the instance once its deployed
![](/image/instance.JPG)

A quick glance at the dependencies reveals a large number of vulnerabilities.

https://github.com/rails/rails/security/advisories/GHSA-65cv-r6x7-79hv
https://nvd.nist.gov/vuln/detail/CVE-2020-8163
https://github.com/rails/rails/security/advisories/GHSA-cfjv-5498-mph5
https://nvd.nist.gov/vuln/detail/CVE-2020-10663
https://nvd.nist.gov/vuln/detail/CVE-2019-5477
https://nvd.nist.gov/vuln/detail/CVE-2020-7595
https://github.com/rack/rack/security/advisories/GHSA-hrqr-hxpp-chr3
https://nvd.nist.gov/vuln/detail/CVE-2020-8184
https://nvd.nist.gov/vuln/detail/CVE-2020-8161

Besides that, there is not much interesting in the instance and I can't find any evidence o. It does however happen to contain all the source code, dependencies and logs for the service.

## Next steps
Looking into the cloudformer directory, I find all the code and a README file which helps me better understand the architecture, I also get access to all the source code.

```   
|   `-- tasks
  |-- log
  |-- public
  |   |-- images
  |   |-- javascripts
  |   `-- stylesheets
  |-- script
  |-- test
  |   |-- fixtures
  |   |-- functional
  |   |-- integration
  |   |-- performance
  |   `-- unit
  |-- tmp
  |   |-- cache
  |   |-- pids
  |   |-- sessions
  |   `-- sockets
  `-- vendor
      `-- plugins

app
  Holds all the code that's specific to this particular application.

app/controllers
  Holds controllers that should be named like weblogs_controller.rb for
  automated URL mapping. All controllers should descend from
  ApplicationController which itself descends from ActionController::Base.

app/models
  Holds models that should be named like post.rb. Models descend from
  ActiveRecord::Base by default.

app/views
  Holds the template files for the view that should be named like
  weblogs/index.html.erb for the WeblogsController#index action. All views use
  eRuby syntax by default.

app/views/layouts
  Holds the template files for layouts to be used with views. This models the
  common header/footer method of wrapping views. In your views, define a layout
  using the <tt>layout :default</tt> and create a file named default.html.erb.
  Inside default.html.erb, call <% yield %> to render the view using this
  layout.

app/helpers
  Holds view helpers that should be named like weblogs_helper.rb. These are
  generated for you automatically when using generators for controllers.
  Helpers can be used to wrap functionality for your views into methods.

config
  Configuration files for the Rails environment, the routing map, the database,
  and other dependencies.

db
  Contains the database schema in schema.rb. db/migrate contains all the
  sequence of Migrations for your schema.

doc
  This directory is where your application documentation will be stored when
  generated using <tt>rake doc:app</tt>

lib
  Application specific libraries. Basically, any kind of custom code that
  doesn't belong under controllers, models, or helpers. This directory is in
  the load path.

public
  The directory available for the web server. Contains subdirectories for
  images, stylesheets, and javascripts. Also contains the dispatchers and the
  default HTML files. This should be set as the DOCUMENT_ROOT of your web
  server.

script
  Helper scripts for automation and generation.

test
  Unit and functional tests along with fixtures. When using the rails generate
  command, template test files will be generated for you and placed in this
  directory.

vendor
  External libraries that the application depends on. Also includes the plugins
  subdirectory. If the app has frozen rails, those gems also go here, under
  vendor/rails/. This directory is in the load path.
  ```

This overview will be useful for Part II where I will attempt to dig deeper into the web application security for cloudformer. Stay tuned for more. If its possible to gain control over the instance, the most useful permission would be to enumerate all S3 buckets and try to exfiltrate the content.