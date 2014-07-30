---
layout: post
title: "Installing scikit-learn on Amazon EC2"
comments: true
date: 2012-12-07 5:05
category: blog
tags: [python, scipy, numpy, scikit-learn, ec2]
modified: 2014-01-28
share: true
---

*(Jan 28 2014): Updated installation to conform to the recommended virtualenv source install.*

I've been using [scikit-learn](http://scikit-learn.org) over the past few weeks on a project.
While developing and analyzing the data I just needed to get work done without the hassle of a complex installation, the Ubuntu image on EC2 provided just that.
Now that the project is ready to be deployed, I need to install scikit-learn on the default Amazon Linux AMI.
As I learned, installing scikit-learn is not trivial.
It only has two dependencies, but those dependencies have dependencies and you have to sift through documentation
of at least 5 packages to truly understand what what is needed to install and in what order.
So I decided to brush up on my writing skills, dust off the old blog, and pen a simple guide that I can reference later.
I'll explain what the scikit-learn dependencies are and how to install them on the Amazon Linux AMI, specifically [image ami-1624987f](http://aws.amazon.com/amazon-linux-ami/).

***requirements***

First we need Python version 2.6 or greater installed.
The main two requirements are [NumPy](http://www.numpy.org) and [SciPy](http://www.scipy.org).
NumPy and SciPy each have their dependencies which are listed below.

1. Numpy
    * c compiler (gcc)
    * fortran compiler (gfortran)
    * python header files (2.4.x - 3.2.x)
    * Strongly recommended BLAS or LAPACK
2. Scipy
    * Numpy
    * Complete LAPACK library

<!-- more -->

Here is where it got confusing for me. NumPy optionally requires (very strongly recommended from what I can tell) BLAS or LAPACK.
SciPy requires LAPACK, but NumPy does not (If BLAS is installed).
By deduction, it seems the sensible choice is to install LAPACK, which both can use, and we're all set.

Not so fast, almost all the documentation says to install ATLAS as a substitute for BLAS. (Where did ATLAS show up in all this?)
It's also recommended to install an optimized LAPACK with a machine specific BLAS library.
What does that all mean?
If you're really interested you can read my attempt to figure all this out at the bottom of this post.
For now, let's just get down to installing all these dependencies and start using scikit-learn.

The really daring can run the [full install script](https://gist.github.com/dacamo76/4780765) directly from the gist

~~~ bash
curl https://gist.github.com/dacamo76/4780765/raw/36acfb10aba554a7738d2fea11d15a31dd8f3a0d/scikit-learn-install.sh | sh
~~~

I'll continue with an explanation of each step in the gist.
Let's install ATLAS, LAPACK, the Python header files, a c++ compiler.

~~~ bash
[ec2-user@ip-10-99-17-223 ~]$ sudo yum install gcc-c++ python27-devel atlas-sse3-devel lapack-devel
~~~

lapack-devel depends on blas-devel which in turn depends on the fortran compiler, so they both pulled in automatically. 
Next install virtualenv and create a virtual python install to keep all our packages separate from the default machine install.


~~~ bash
[ec2-user@ip-10-99-17-223 ~]$ wget https://pypi.python.org/packages/source/v/virtualenv/virtualenv-1.11.2.tar.gz
[ec2-user@ip-10-99-17-223 ~]$ tar xzf virtualenv-1.11.2.tar.gz
[ec2-user@ip-10-99-17-223 ~]$ python27 virtualenv-1.11.2/virtualenv.py sk-learn
New python executable in sk-learn/bin/python27
Also creating executable in sk-learn/bin/python
Installing setuptools, pip...done.
~~~
Activate the new Python 2.7 virtualenv.

~~~ bash
[ec2-user@ip-10-99-17-223 ~]$ . sk-learn/bin/activate
(sk-learn)[ec2-user@ip-10-99-17-223 ~]$
~~~

Now install numpy, it should find and use the optmized linear algebra libraries.

~~~ bash
(sk-learn)[ec2-user@ip-10-99-17-223 ~]$ pip install numpy
~~~

Verify that NumPy found the optmized linear algebra libraries.

~~~ bash
(sk-learn)[ec2-user@ip-10-99-17-223 ~]$ python -c "import numpy; numpy.show_config()"
atlas_threads_info:
    libraries = ['lapack', 'ptf77blas', 'ptcblas', 'atlas']
    library_dirs = ['/usr/lib64/atlas-sse3']
    define_macros = [('ATLAS_INFO', '"\\"3.8.4\\""')]
    language = f77
    include_dirs = ['/usr/include']
blas_opt_info:
    libraries = ['ptf77blas', 'ptcblas', 'atlas']
    library_dirs = ['/usr/lib64/atlas-sse3']
    define_macros = [('ATLAS_INFO', '"\\"3.8.4\\""')]
    language = c
    include_dirs = ['/usr/include']
atlas_blas_threads_info:
    libraries = ['ptf77blas', 'ptcblas', 'atlas']
    library_dirs = ['/usr/lib64/atlas-sse3']
    define_macros = [('ATLAS_INFO', '"\\"3.8.4\\""')]
    language = c
    include_dirs = ['/usr/include']
lapack_opt_info:
    libraries = ['lapack', 'ptf77blas', 'ptcblas', 'atlas']
    library_dirs = ['/usr/lib64/atlas-sse3']
    define_macros = [('ATLAS_INFO', '"\\"3.8.4\\""')]
    language = f77
    include_dirs = ['/usr/include']
lapack_mkl_info:
  NOT AVAILABLE
blas_mkl_info:
  NOT AVAILABLE
mkl_info:
  NOT AVAILABLE
~~~

If you don't see output for atlas_threads_info, blas_opt_info, atlas_blas_threads_info, or lapack_opt_info then
NumPy did not find the ATLAS libraries.
If you're seeing output similar to the following it's probably not what you want.

~~~ bash
(sk-learn)[ec2-user@ip-10-99-17-223 ~]$ python -c "import numpy; numpy.show_config()"
blas_info:
  NOT AVAILABLE
lapack_info:
  NOT AVAILABLE
atlas_threads_info:
  NOT AVAILABLE
blas_src_info:
  NOT AVAILABLE
lapack_src_info:
  NOT AVAILABLE
atlas_blas_threads_info:
  NOT AVAILABLE
lapack_opt_info:
  NOT AVAILABLE
blas_opt_info:
  NOT AVAILABLE
atlas_info:
  NOT AVAILABLE
lapack_mkl_info:
  NOT AVAILABLE
blas_mkl_info:
  NOT AVAILABLE
atlas_blas_info:
  NOT AVAILABLE
mkl_info:
  NOT AVAILABLE
~~~

NumPy is installed but will not use the ATLAS libraries.
At this point it's best to start over from step 1 and make sure atlas-sse3-devel and lapack-devel are installed.
I recommend removing the sk-learn (virtualenv) directory and creating the virtualenv again, this makes sure NumPy
gets re-installed from scratch and the old version is not lingering around to confuse things.

Once NumPy is successfully installed and linked to the ATLAS libraries, continue by installing SciPy and scikit-learn.

~~~ bash
(sk-learn)[ec2-user@ip-10-99-17-223 ~]$ pip install scipy
(sk-learn)[ec2-user@ip-10-99-17-223 ~]$ pip install scikit-learn
~~~

scikit-learn is now installed!!
Let's run the scikit-learn tests to verify everything is installed correctly.
Install nose

~~~ bash
(sk-learn)[ec2-user@ip-10-99-17-223 ~]$ pip install nose
~~~

and in a directory outside the source run the tests.

~~~ bash
(sk-learn)[ec2-user@ip-10-99-17-223 ~]$ nosetests sklearn --exe
~~~

That's it. We now have scikit-learn installed and ready to go on EC2.

---
This is where I ramble a little as I try to keep for future reference what all these acronyms mean.

[BLAS](http://www.netlib.org/blas/) (Basic Linear Algebra Subprograms)

:   Routines that provide standard building blocks for performing basic vector and matrix operations

[LAPACK](http://www.netlib.org/lapack/) (Linear Algebra PACKage)

:   Routines for solving systems of simultaneous linear equations, least-squares solutions of linear systems of equations, eigenvalue problems, and singular value problems

[ATLAS](http://math-atlas.sourceforge.net) (Automatically Tuned Linear Algebra Software)

:   Complete optimized implementation of the BLAS API and a small subset of the LAPACK API

I pick up at the point where we know we need to install BLAS and LAPACK as a prerequisite to installing SciPy.
ATLAS is the recommended BLAS implementation as it provides a machine optimized complete implementation of the BLAS libraries.
We still need to install a full version of LAPACK for SciPy to be satisfied since ATLAS only provides a small subset of the LAPACK API.

It seems so simple now that I write it down, but I had to hunt through various message groups for it all to make sense.
