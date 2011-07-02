Doctrine Annotations
====================

The Doctrine Common annotations library was born from a need in the
`Doctrine2 ORM <http://www.doctrine-project.org/projects/orm>`_ to
allow the mapping information to be specified as metadata embedded
in the class files, on properties and methods. The library is
independent and can be used in your own libraries to implement doc
block annotations.

Introduction
------------

There are several different approaches to handling annotations in PHP. Doctrine Annotations
maps docblock annotations to PHP classes. Because not all docblock annotations are used
for metadata purposes a filter is applied to ignore or skip classes that are not Doctrine annotations.

Take a look at the following code snippet:

.. code-block :: php

    <?php
    namespace MyProject\Entities;

    use Doctrine\ORM\Mapping AS ORM;
    use Symfony\Component\Validation\Constraints AS Assert;

    /**
     * @author Benjamin Eberlei
     * @ORM\Entity
     * @MyProject\Annotations\Foobarable
     * @Gedmo:Timestampable
     */
    class User
    {
        /** 
         * @ORM\Id @ORM\Column @ORM\GeneratedValue 
         * @dummy
         * @var int
         */
        private $id;

        /**
         * @ORM\Column(type="string")
         * @Assert\NotEmpty
         * @Assert\Email
         * @var string
         */
        private $email;
    }

In this snippet you can see a variety of different docblock annotations:

- Documentation annotations such as @var and @author. These annotations are on a blacklist and never considered for throwing an exception due to wrongly used annotations.
- Annotations imported through use statements. The statement "use Doctrine\ORM\Mapping AS ORM" makes all classes under that namespace available as @ORM\ClassName. Same goes for the import of "Assert".
- The @dummy annotation. It is not a documentation annotation and not blacklisted. For Doctrine Annotations it is not entirely clear how to handle this annotation. Depending on the configuration an exception (unknown annotation) will be thrown when parsing this annotation.
- The fully qualified annotation @MyProject\Annotations\Foobarable. This is transformed directly into the given class name.
- The aliased Annotation @Gedmo:Timestampable. An alias resolves to a namespace. 

How are these annotations loaded? From looking at the code you could guess that the ORM Mapping, Assert Validation and the fully qualified annotation can just be loaded using
the defined PHP autoloaders. This is not the case however: For error handling reasons every check for class existance inside the AnnotationReader sets the second parameter $autoload
of ``class_exists($name, $autoload)`` to false. To work flawlessly the AnnotationReader requires silent autoloaders which many autoloaders are not. Silent autoloading is NOT
part of the [PSR-0 specification](http://groups.google.com/group/php-standards/web/psr-0-final-proposal) for autoloading.

This is why Doctrine Annotations uses its own autoloading mechanism through a global registry. If you are wondering about the annotation registry being global,
there is no other way to solve the architectural problems of autoloading annotation classes in a straightforward fashion. Additionally if you think about PHP
autoloading then you recognize it is a global as well.

To anticipate the configuration section, making the above PHP class work with Doctrine Annotations requires this setup:

    <?php
    use Doctrine\Common\Annotations\AnnotationReader;
    use Doctrine\Common\Annotations\AnnotationRegistry;

    AnnotationRegistry::registerFile("/path/to/doctrine/lib/Doctrine/ORM/Mapping/Driver/DoctrineAnnotations.php");
    AnnotationRegistry::registerAnnotationNamespace("Symfony\Component\Validator\Constraint", "/path/to/symfony/src");
    AnnotationRegistry::registerAnnotationNamespace("MyProject\Annotations", "/path/to/myproject/src");

    $reader = new AnnotationReader();
    $reader->setIgnoreNotImportedAnnotations( true );

The second block with the annotation registry calls registers all the three different annotation namespaces that are used.
Doctrine saves all its annotations in a single file, that is why ``AnnotationRegistry#registerFile`` is used in contrast to
``AnnotationRegistry#registerAnnotationNamespace`` which creates a PSR-0 compatible loading mechanism for class to file names.

In the third block creates the AnnotationReader instance and sets the error reporting to ignore not imported exceptions.
This prevents you from finding typos in annotations, however it also allows you to use arbitrary annotations without failure.
Enabling or disabling this variable is a double-edged sword. Ignoring not imported annotations prevents validation of annotations,
however it also liberate you from the strict requirements on your docblocks, mainly that unknown annotations will make your code fail.
Setting this variable is necessary in our example case, otherwise @dummy would throw an exception while parsing the docblock
of ``MyProject\Entities\User#id``.

Setup and Configuration
-----------------------

To use the annotations library is simple, you just need to create a new ``AnnotationReader`` instance:

.. code-block :: php

    <?php
    $reader = new \Doctrine\Common\Annotations\AnnotationReader();

This creates a simple  annotation reader with no caching other than in memory (in php arrays).
Since parsing docblocks can be expensive you should cache this process by using
a caching reader.

You can use a file caching reader:

.. code-block :: php

    <?php
    use Doctrine\Common\Annotations\FileCacheReader;
    use Doctrine\Common\Annotations\AnnotationReader;

    $reader = new FileCacheReader(
        new AnnotationReader(),
        "/path/to/cache",
        $debug = true
    );

If you set the debug flag to true the cache reader will check for changes in the original files, which
is very important during development. If you don't set it to true you have to delete the directory to clear the cache.
This gives faster performance, however should only be used in production, because of its inconvenience
during development.

You can also use one of the ``Doctrine\Common\Cache\Cache`` cache implementations to cache the annotations:

<?php

    <?php
    use Doctrine\Common\Annotations\AnnotationReader;
    use Doctrine\Common\Annotations\CachedReader;
    use Doctrine\Common\Cache\ApcCache;

    $reader = new CachedReader(
        new AnnotationReader(),
        new ApcCache(),
        $debug = true
    );

The debug flag is used here as well to invalidate the cache files when the PHP class with annotations changed
and should be used during development.

Registering Annotations
~~~~~~~~~~~~~~~~~~~~~~~

As explained in the Introduction Doctrine Annotations uses its own autoloading mechanism to determine if a
given annotation has a corresponding PHP class that can be autoloaded.

Default Namespace
~~~~~~~~~~~~~~~~~

If you don't want to specify the fully qualified class name you can
set the default annotation namespace using the
``setDefaultAnnotationNamespace()`` method. The following is an
example where we specify the fully qualified class name for the
annotation:

.. code-block :: php

    <?php
    /** @MyCompany\Annotations\Foo */
    class Test
    {
    }

To shorten the above code you can configure the default namespace
to be ``MyCompany\Annotations``:

.. code-block :: php

    <?php
    $reader->setDefaultAnnotationNamespace('MyCompany\Annotations\\');

Now it can look something like:

.. code-block :: php

    <?php
    /** @Foo */
    class Test
    {
    }

A little nicer looking!

Namespace Aliases
~~~~~~~~~~~~~~~~~

Again to save you from having to specify the fully qualified class
name you can set an alias for a namespace of annotation classes:

.. code-block :: php

    <?php
    $reader->setAnnotationNamespaceAlias('MyCompany\Annotations\\', 'my');

So now you could do something like this:

.. code-block :: php

    <?php
    /** @my:Foo */
    class Test
    {
    }

Again, a bit nicer looking than the fully qualified class name!

Annotation Creation
~~~~~~~~~~~~~~~~~~~

If you want to customize the creation process of the annotation
class instances then you have two options, you can sub-class the
``Doctrine\Common\Annotations\Parser`` class and override the
``newAnnotation()`` method:

.. code-block :: php

    <?php
    class MyParser extends Parser
    {
        protected function newAnnotation($name, array $values)
        {
            return new $name($values);
        }
    }

The other option is to use the ``setAnnotationCreationFunction()``
method to specify a closure to execute:

.. code-block :: php

    <?php
    $reader->setAnnotationCreationFunction(function($name, array $values) {
        return new $name($values);
    });

Usage
-----

Using the library API is simple. First lets define some annotation
classes:

.. code-block :: php

    <?php
    namespace MyCompany\Annotations;
    
    class Foo extends \Doctrine\Common\Annotations\Annotation
    {
        public $bar;
    }
    
    class Bar
    {
        public $foo;
    }

Now to use the annotations you would just need to do the
following:

.. code-block :: php

    <?php
    /**
     * @Foo(bar="foo")
     * @Bar(foo="bar")
     */
    class User
    {
    }

Now we can write a script to get the annotations above:

.. code-block :: php

    <?php
    $reflClass = new ReflectionClass('User');
    $classAnnotations = $reader->getClassAnnotations($reflClass);
    echo $classAnnotations['MyCompany\Annotations\Foo']->bar; // prints foo
    echo $classAnnotations['MyCompany\Annotations\Foo']->foo; // prints bar

You have a complete API for retrieving annotation class instances
from a class, property or method docblock:


-  getClassAnnotations(ReflectionClass $class)
-  getClassAnnotation(ReflectionClass $class, $annotation)
-  getPropertyAnnotations(ReflectionProperty $property)
-  getPropertyAnnotation(ReflectionProperty $property, $annotation)
-  getMethodAnnotations(ReflectionMethod $method)
-  getMethodAnnotation(ReflectionMethod $method, $annotation)

.. warning ::

    The AnnotationReader works and caches under the
    assumption that all annotations of a doc-block are processed at
    once. That means that annotation classes that do not exist and
    aren't loaded and cannot be autoloaded (using
    setAutoloadAnnotationClasses()) would never be visible and not
    accessible if a cache is used unless the cache is cleared and the
    annotations requested again, this time with all annotations
    defined.



