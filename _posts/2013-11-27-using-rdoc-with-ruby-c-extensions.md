---
layout: post
title: Using RDoc with Ruby C Extensions
date: 2013-11-27 19:01:46.000000000 -08:00
status: published
type: post
---
While building a native C extension for Ruby, I discovered that RDoc
will occasionally refuse to document valid code and emit no particular
warnings or error messages when it chooses to do so. It turns out that
if the class definitions exist across multiple C sources, RDoc will
skip over parsing documentation if it cannot locate any documentation
for the dependent classes or modules.

This issue may show up if you have multiple classes defined under a
module, and each class and module in a separate C source file. RDoc has
no knowledge of the structure of C programs (nor the ability to use the
C pre-processor) so it isn't capable of understanding some valid
constructs and it requires a kick in the pants to do so.

In your `extconf.rb`, you will need to add a compiler flag before the
call to `create_makefile`:

{% highlight ruby %}
$defs.push('-DRDOC_CAN_PARSE_DOCUMENTATION=0')
{% endhighlight %}

Then, in the initialization function for each class, you need to add a
bit of dummy code before RDoc will understand how to read the code:

{% highlight c %}
void Init_MyExtension_MyModule_MyClass(void)
{
#if RDOC_CAN_PARSE_DOCUMENTATION
    mMyModule = rb_define_module("MyModule");
#endif
    cMyClass = rb_define_class_under(mMyModule, "MyClass", rb_cObject);
}
{% endhighlight %}

Now, RDoc will see the module definition and parse documentation as usual
while the C pre-processor will omit this code from the compilation unit.
