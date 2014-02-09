---
layout: post
title: Transitory Domain Objects
date: 2009-05-30 20:53:46.000000000 +02:00
categories:
  - seb
tags:
  - domain-driven-design
  - software-architecture
---
A common problem with DDD is the injection of services to your domain model. Sometimes your domain relies on external services to do it's job. You could do that by injecting your services directly to your entities using <a href="http://www.nhforge.org/doc/nh/en/index.html#manipulatingdata-interceptors">NHibernate Interceptors</a> or <a href="http://rogeralsing.com/2009/05/30/entity-framework-4-entity-dependency-injection/">ObjectStateManager for Entity Framework v4</a>.

There are many design issues with the POCOness of Entities when you keep references to external services within the Entities themselves. The reference itself is (usually) infrastructure and not really a persistence concern.

## Double Dispatch, Specifications and Services
The double dispatch pattern seems to be a popular approach. A better solution seem to be to move the logic in front of the Entites. Usually people seem to solve this by moving logic to <a href="http://devlicio.us/blogs/casey/archive/2009/02/17/ddd-services.aspx">services</a> or even <a href="http://devlicio.us/blogs/casey/archive/2009/03/02/ddd-the-specification-pattern.aspx">specifications</a>.

Moving domain logic to services is a big no, no. That's a gateway to anemic domain models and bloated service implementations. Services should be a last resort for external concerns and should probably have a solid anti-corruption layer.

The double dispatch pattern is a pain, ugly and introduces lots of references to services where the ubiquitous language doesn't dictate it. 

The specification pattern is particularly ugly because that's (usually) not how a domain expert would refer to the issue. We are violating the ubiquitous language.

## Transitory Domain Objects
Recently I've started introducing unpersisted classes to my domain models. If you think about it, many domain models have transitory terms and concerns that are not really persisted.

Imagine that your domain model consists of an archive of home photography. Let's call them Photos. Now, you want to work with a couple of them. You pick out all the ones that have a red lavish hue and start organizing, labeling them or other operations. Now you have a set of Photos.

You could claim that it is a UI or Controller concern. Given the right bounded context, that set of photos IS A Domain Concern! Your domain could have domain specific restrictions and operations occurring on those sets of photos. You can think about them as a workspace or extended units of work.

Now this set isn't persisted. It's not an entity, it's not a value object. Your entities can't refer to it. This transitory logic lies infront of your entities. Since it's transitory it also means that it can contain references to repositories and external services. It makes reference management much easier.

Now we can change out our specification and double dispatch patterns:

{% highlight c# %}
var redishPhotoList = photoRepository.Find(
  new HueSpecificiation(colorDetectorService, Color.Red)
);
foreach(var photo in redishPhotoList){
  //checks...
  photo.MarkWithMetaData("RED", metaDataService);
  //contraints...
}
{% endhighlight %}

To something more domain specific:

{% highlight c# %}
var photoSet = new PhotoSet(photoRepository, colorDetectorService, metaDataService);
photoSet.UsingOnly(Color.Red).MarkWithMetaData("RED");
{% endhighlight %}

We now have a domain object that we can easily pass around our application.

When you think about it you're probably already using this pattern either as helpers or as "services". But making the clear distinction that this is 1) A Domain Concern. 2) Temporary. Makes it easier to place your logic and apply constraints.

*Achieving pure POCO is a pain from an infrastructure perspective but it's worth it once it's in place. I should be able to pass it to and from Db4O without any infrastructure concerns. Then you have a clear and solid domain model.*
