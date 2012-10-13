---
layout: post
title: "Java Modeling Language"
date: 2012-10-12 16:40
comments: true
categories: 
---

Java Modeling Language is a Design by Contract(DBC) specification language for Java programs. DBC is a programming methodology, 
which was introduced by B. Meyer and implemented in Eiffel programming language. The idea is pretty simple – a program component must do exactly 
what is described in the contract. Hence, a user of the component may learn about it using the contract, the implementation of the component 
might be different by must follow the contract. In Object Oriented languages a component is a class and a contract regulates the state of an object as well as behavior.
Using JML a programmer may specify the contract for methods and attributes. It is written as a comment in the class and 
translated by the JML compiler into the Java code. 
<!--more-->
Lets have a look at the example:
``` java
class Building {

  int floorsNumber;
  static final int MAX_FLOOR = 300;

  //@ invariant getFloorsNumber() >= 0 && 
  //@	getFloorsNumber() < MAX_FLOOR;

  //@ requires newFloors < MAX_FLOOR;
  //@ ensures getFloorNumber () < MAX_FLOOR;
  void addNewFloors(int newFloors) {
	floorsNumber += newFloors;
  }

  //@ pure
  int getFloorNumber() {
	return floorsNumber;
  }

}
```

In this example, Building’s contract for the state specifies that the number of floors must be non-negative and not exceed the specified limit.
The keyword invariant is used to specify a contract for the object state. Method addNewFloors has prerequrement (“requires” keyword) that newFloors 
is less than the limit and postreqirement (“ensures” keyword) that floorsNumber is less than the limit. The keyword “pure”(near getFloorNumber) 
tells to the JML translator that this method doesn’t have side effects and might be used in JML specifications.

As you may see the syntax of the JML specification is comprehensible and can be understood even without knowledge in the domain. 
The JML code is kind of developed assertions ensures that some assumptions written as predicates are true.

The JML syntax is quite expressive so a programmer can write sophisticated predicates with loops, sums and other constructions. 
For instance, the JML specification for the method sorting array may look like that:

``` java 
/*@ ensures 
(\forall int i; 0 <= i && i < a.length-1;
a[i] < a[i+1]);
@*/
byte[] sort(byte[] a);
```

If you are interested you may read about syntax of JML at the <a href="http://www.eecs.ucf.edu/~leavens/JML/papers.shtml">JML web site</a>.

Now I will write about technical problems with JML. There are several JML compilers available – JML 5.4, Esc, OpenJML. The problem is 
that all of them are developed more as a proof of concept than really working application.  Hence, they are buggy and are not well supported. 
The JML 5.4 is the most reliable one yet it works only with Java 1.4. The OpenJML should have substituted JML 5.4 but at the current 
moment it is just a prototype. The sad thing about it is the development was stopped one year ago.
If you want to try JML5.4, download <a href="http://www2.icorsi.ch/mod/resource/view.php?id=54421">it</a>, add environment variable JML = \<path to JML\>.
To compile your application:
```
java -cp "$JML/bin/jml-release.jar:$JML/specs:." org.jmlspecs.jmlrac.Main "<your *.java>"
```
To run it:
```
java -cp "${JML}/bin/jmlruntime.jar:." $*
```

I think, it could be a good master thesis to make a working application out of OpenJML. If you think that it is not scientific enough, 
there are several PhD dissertations at leading CS universities dedicated to development of DbC compilers for various languages. 
Most of these languages are never used in industry. At the same time JML is used by at least students studying verification and similar courses so this work would be, not doubt, useful.
