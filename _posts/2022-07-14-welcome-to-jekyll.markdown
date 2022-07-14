---
layout: post
title:  "First post"
date:   2022-07-14 11:48:52 +0200
categories: blog
---
Alright, so this is the first, test entry on the blog. Welcome, more content to
follow soon. Meanwhile I'll just put a random code snippet here to have a cheat
sheet for later. And also to see how it looks like rendered.

{% highlight golang %}
/// [Actuator]
// Actuator controls machines on a specific infrastructure. All
// methods should be idempotent unless otherwise specified.
type Actuator interface {
	// Create the machine.
	Create(context.Context, *machinev1.Machine) error
	// Delete the machine. If no error is returned, it is assumed that all dependent resources have been cleaned up.
	Delete(context.Context, *machinev1.Machine) error
	// Update the machine to the provided definition.
	Update(context.Context, *machinev1.Machine) error
	// Checks if the machine currently exists.
	Exists(context.Context, *machinev1.Machine) (bool, error)
}
{% endhighlight %}
