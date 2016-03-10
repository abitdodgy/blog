---
layout: post
title: Activity Feed
---

I recently built our activity feed at Maestru from scratch, and it was a great learning experience. In this tutorial I will show you how I did it.

Our activity feed is complex and has several layers. It tracks a growing collection of 15 objects, and has different visibility and permission requirements. To keep this tutorial as short as possible, I will not cover everything. But I will explore each of the feed's major charateristics in sufficient detail.

To make this tutorial easier to understand and follow, I will start small and use progressive enhancement to add more features and objects to the feed. Otherwise there will be too many concepts to explain at once.

## Before you get started

The tutorial app is modeled after our application Maestru. For the sake of brevity, I'll leave it up to you to use version control. You can find the [source code for this app][2] on Github.

I have include only some tests in this tutorial, but the Github repo includes all of them.

### Outline

1. Introduction
2. What we are building
2. Requirements
3. Scaling the feed

## Setup

I will start with three classes: Workspaces, users, memberships. A workspace has many members through memberships.

[Entity Relationship Diagram showing workspaces, users, and memberships]

To start, create a new Rails app[^1].

{% highlight text %}
rails new activity-feed --database postgresql
cd activity-feed
rake db:create
{% endhighlight %}

Now create the objects.

{% highlight text %}
rails g model user name
rails g model workspace name
rails g model membership user:belongs_to workspace:belongs_to role:integer
{% endhighlight %}

Open the `memberships` migration and make a small change to the `role` column. The `roles` column is of type integer because I will define roles using an [enum][1]. Add a default value of `1` to the column.

{% highlight ruby %}
class CreateMemberships < ActiveRecord::Migration
  def change
    create_table :memberships do |t|
      # ...
      t.integer :role, null: false, default: 1
      # ...
    end
  end
end
{% endhighlight %}

Migrate the database, and add the associations.

{% highlight text %}
rake db:migrate
{% endhighlight %}

{% highlight ruby %}
class User < ActiveRecord::Base
  has_many :memberships
  has_many :workspaces, through: :memberships
end

class Workspace < ActiveRecord::Base
  has_many :memberships
  has_many :users, through: :memberships
end

class Membership < ActiveRecord::Base
  belongs_to :user
  belongs_to :workspace
end
{% endhighlight %}

Now you are ready to add the activities table.

## Activities

Activities should be scoped to the workspace. Naturally, an activity belongs to an actor, or a user.





I will include some tests, but only those I feel are important. The complete application will

I have simplified the objects included in this tutorial by removing many fields, but nothing that is relavent or necessary for the feed itself.




## Requirements

Our application has a top level Workspace object. A workspace has many projects. Projects are where we store, edit, and share documents. Projects can be public or private. Public projects are accessible to all members of the workspace. Private projects are accessible to admins, and regular users whom have been given access. Here's a simplified list of the objects we have:

1. Workspace
6. Membership
2. Project
3. Document
4. Comment
5. Upload

### A feed for documents

At the most granular level are individual document feeds. A document feed shows activities such as downloads, uploads of new versions, changes to status and labels. This is very close to Github's issue feed.

### A feed for projects

The project feed is one level higher than the document feed. It must show activities that result from changes to the project, new document uploads, and many but not all of the activities shown in the document feed for documents belonging to the project.

existing documents  that belong to the project (many of )such as versioning, status and other changes. Some of the items in this feed will be shown on the workspace feed.

### General feed for the workspace

Our activity feed must track changes to the workspace and its descendent objects. We need a high level feed to display workspace activity. This includes creating and editing areas, as well as adding and removing new members. It must act as an aggregator for projects' feeds that belong to the workspace, which we will get to in a minute.

All activities in this feed must be visible to admins. Activities that belong to public projects must be visible to everyone. Activities that belong to private projects must be visible to admins and users whom have been granted access.



[1]: http://edgeapi.rubyonrails.org/classes/ActiveRecord/Enum.html
[2]: https://github.com/abitdodgy/activity-feed

[^1]: You can use any database you want, but in this tutorial I will use Postgresql.

I looked at the [PublicActivity] gem, but we have very specific requirements and I did not want to risk tinkering or fighting with an external library to make it fit into our requirements. It's important to have some understanding of our application's structure before we continue.

