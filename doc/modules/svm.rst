.. _svm:

=======================
Support Vector Machines
=======================

.. currentmodule:: sklearn.svm

**Support vector machines (SVMs)** are a set of supervised learning
methods used for :ref:`classification <svm_classification>`,
:ref:`regression <svm_regression>` and :ref:`outliers detection
<svm_outlier_detection>`.

The advantages of support vector machines are:

    - Effective in high dimensional spaces.

    - Still effective in cases where number of dimensions is greater
      than the number of samples.

    - Uses a subset of training points in the decision function (called
      support vectors), so it is also memory efficient.

    - Versatile: different :ref:`svm_kernels` can be
      specified for the decision function. Common kernels are
      provided, but it is also possible to specify custom kernels.

The disadvantages of support vector machines include:

    - If the number of features is much greater than the number of
      samples, the method is likely to give poor performances.

    - SVMs do not directly provide probability estimates, these are
      calculated using five-fold cross-validation, and thus
      performance can suffer.

The support vector machines in scikit-learn support both dens
(``numpy.ndarray`` and convertible to that by ``numpy.asarray``) and
sparse (any ``scipy.sparse``) sample vectors as input. However, to use
an SVM to make predictions for sparse data, it must have been fit on such
data. For optimal performance, use C-ordered ``numpy.ndarray`` (dense) or
``scipy.sparse.csr_matrix`` (sparse) with ``dtype=float64``.

In previous versions of scikit-learn, sparse input support existed only
in the ``sklearn.svm.sparse`` module which duplicated the ``sklearn.svm``
interface. This module still exists for backward compatibility, but is
deprecated and will be removed in scikit-learn 0.12.

.. TODO: add reference to probability estimates

.. _svm_classification:

Classification
==============

:class:`SVC`, :class:`NuSVC` and :class:`LinearSVC` are classes
capable of performing multi-class classification on a dataset.


.. figure:: ../auto_examples/svm/images/plot_iris_1.png
   :target: ../auto_examples/svm/plot_iris.html
   :align: center


:class:`SVC` and :class:`NuSVC` are similar methods, but accept
slightly different sets of parameters and have different mathematical
formulations (see section :ref:`svm_mathematical_formulation`). On the
other hand, :class:`LinearSVC` is another implementation of Support
Vector Classification for the case of a linear kernel. Note that
:class:`LinearSVC` does not accept keyword 'kernel', as this is
assumed to be linear. It also lacks some of the members of
:class:`SVC` and :class:`NuSVC`, like support\_.

As other classifiers, :class:`SVC`, :class:`NuSVC` and
:class:`LinearSVC` take as input two arrays: an array X of size
[n_samples, n_features] holding the training samples, and an array Y
of integer values, size [n_samples], holding the class labels for the
training samples::


    >>> from sklearn import svm
    >>> X = [[0, 0], [1, 1]]
    >>> Y = [0, 1]
    >>> clf = svm.SVC()
    >>> clf.fit(X, Y)  # doctest: +NORMALIZE_WHITESPACE
    SVC(C=None, cache_size=200, class_weight=None, coef0=0.0, degree=3,
    gamma=0.5, kernel='rbf', probability=False, scale_C=True, shrinking=True,
    tol=0.001, verbose=False)

After being fitted, the model can then be used to predict new values::

    >>> clf.predict([[2., 2.]])
    array([ 1.])

SVMs decision function depends on some subset of the training data,
called the support vectors. Some properties of these support vectors
can be found in members `support_vectors_`, `support_` and
`n_support`::

    >>> # get support vectors
    >>> clf.support_vectors_
    array([[ 0.,  0.],
           [ 1.,  1.]])
    >>> # get indices of support vectors
    >>> clf.support_ # doctest: +ELLIPSIS
    array([0, 1]...)
    >>> # get number of support vectors for each class
    >>> clf.n_support_ # doctest: +ELLIPSIS
    array([1, 1]...)

.. _svm_multi_class:

Multi-class classification
--------------------------

:class:`SVC` and :class:`NuSVC` implement the "one-against-one"
approach (Knerr et al., 1990) for multi- class classification. If
n_class is the number of classes, then n_class * (n_class - 1)/2
classifiers are constructed and each one trains data from two classes::

    >>> X = [[0], [1], [2], [3]]
    >>> Y = [0, 1, 2, 3]
    >>> clf = svm.SVC()
    >>> clf.fit(X, Y) # doctest: +NORMALIZE_WHITESPACE
    SVC(C=None, cache_size=200, class_weight=None, coef0=0.0, degree=3,
    gamma=1.0, kernel='rbf', probability=False, scale_C=True, shrinking=True,
    tol=0.001, verbose=False)
    >>> dec = clf.decision_function([[1]])
    >>> dec.shape[1] # 4 classes: 4*3/2 = 6
    6

On the other hand, :class:`LinearSVC` implements "one-vs-the-rest"
multi-class strategy, thus training n_class models. If there are only
two classes, only one model is trained::

    >>> lin_clf = svm.LinearSVC()
    >>> lin_clf.fit(X, Y) # doctest: +NORMALIZE_WHITESPACE
    LinearSVC(C=None, class_weight=None, dual=True, fit_intercept=True,
    intercept_scaling=1, loss='l2', multi_class='ovr', penalty='l2',
    scale_C=True, tol=0.0001, verbose=0)
    >>> dec = lin_clf.decision_function([[1]])
    >>> dec.shape[1]
    4

See :ref:`svm_mathematical_formulation` for a complete description of
the decision function.

Note that the :class:`LinearSVC` also implements an alternative multi-class
strategy, the so-called multi-class SVM formulated by Crammer and Singer, by
using the option "multi_class='crammer_singer'". This method is consistent,
which is not true for one-vs-rest classification.
In practice, on-vs-rest classification is usually preferred, since the results
are mostly similar, but the runtime is significantly less.

For "one-vs-rest" :class:`LinearSVC` the attributes ``coef_`` and ``intercept_``
have the shape ``[n_class, n_features]`` and ``[n_class]`` respectively.
Each row of the coefficients corresponds to one of the ``n_class`` many
"one-vs-rest" classifiers and simliar for the interecepts, in the
order of the "one" class.

In the case of "one-vs-one" :class:`SVC`, the layout of the attributes
is a little more involved. In the case of having a linear kernel,
The layout of ``coef_`` and ``intercept_`` is similar to the one
described for :class:`LinearSVC` described above, except that
the shape of ``coef_`` is ``[n_class * (n_class - 1) / 2``,
corresponding to as many binary classifiers. The order for classes
0 to n is "0 vs 1", "0 vs 2" , ... "0 vs n", "1 vs 2", "1 vs 3", "1 vs n", . .
. "n-1 vs n".

The shape of ``dual_coef_`` is ``[n_class-1, n_SV]`` with
a somewhat hard to grasp layout.
The columns correspond to the support vectors involved in any
of the ``n_class * (n_class - 1) / 2`` "one-vs-one" classifiers.
Each of the support vectors is used in ``n_class - 1`` classifiers.
The ``n_class - 1`` entries in each row correspond to the dual coefficients
for these classifiers.

This might be made more clear by an example:

    Consider a three class problem with with class 0 having 3 support vectors
    :math:`v^{0}_0, v^{1}_0, v^{2}_0` and class 1 and 2 having two support
    vectors :math:`v^{0}_1, v^{1}_1` and :math:`v^{0}_1, v^{1}_1` respectively.
    For each support vector :math:`v^{j}_i`, there are 2 dual coefficients.
    Let's call the coefficient of support vector :math:`v^{j}_i` in the
    classifier between classes `i` and `k` :math:`\alpha^{j}_{i,k}`.
    Then ``dual_coef_`` looks like this:

    +------------------------+------------------------+------------------+
    |:math:`\alpha^{0}_{0,1}`|:math:`\alpha^{0}_{0,2}`|Coefficients      |
    +------------------------+------------------------+                  |
    |:math:`\alpha^{1}_{0,1}`|:math:`\alpha^{1}_{0,2}`|for SVs           |
    +------------------------+------------------------+                  |
    |:math:`\alpha^{2}_{0,1}`|:math:`\alpha^{2}_{0,2}`|of class 0        |
    +------------------------+------------------------+------------------+
    |:math:`\alpha^{0}_{1,0}`|:math:`\alpha^{0}_{1,2}`|Coefficients      |
    +------------------------+------------------------+                  |
    |:math:`\alpha^{1}_{1,0}`|:math:`\alpha^{1}_{1,2}`|for SVs of class 1|
    +------------------------+------------------------+------------------+
    |:math:`\alpha^{0}_{2,0}`|:math:`\alpha^{0}_{2,1}`|Coefficients      |
    +------------------------+------------------------+                  |
    |:math:`\alpha^{1}_{2,0}`|:math:`\alpha^{1}_{2,1}`|for SVs of class 2|
    +------------------------+------------------------+------------------+


Unbalanced problems
--------------------

In problems where it is desired to give more importance to certain
classes or certain individual samples keywords ``class_weight`` and
``sample_weight`` can be used.

:class:`SVC` (but not :class:`NuSVC`) implement a keyword
``class_weight`` in the fit method. It's a dictionary of the form
``{class_label : value}``, where value is a floating point number > 0
that sets the parameter C of class ``class_label`` to C * value.

.. figure:: ../auto_examples/svm/images/plot_separating_hyperplane_unbalanced_1.png
   :target: ../auto_examples/svm/plot_separating_hyperplane_unbalanced.html
   :align: center
   :scale: 75


:class:`SVC`, :class:`NuSVC`, :class:`SVR`, :class:`NuSVR` and
:class:`OneClassSVM` implement also weights for individual samples in method
``fit`` through keyword sample_weight.


.. figure:: ../auto_examples/svm/images/plot_weighted_samples_1.png
   :target: ../auto_examples/svm/plot_weighted_samples.html
   :align: center
   :scale: 75


.. topic:: Examples:

 * :ref:`example_svm_plot_iris.py`,
 * :ref:`example_svm_plot_separating_hyperplane.py`,
 * :ref:`example_svm_plot_separating_hyperplane_unbalanced.py`
 * :ref:`example_svm_plot_svm_anova.py`,
 * :ref:`example_svm_plot_svm_nonlinear.py`
 * :ref:`example_svm_plot_weighted_samples.py`,


.. _svm_regression:

Regression
==========

The method of Support Vector Classification can be extended to solve
regression problems. This method is called Support Vector Regression.

The model produced by support vector classification (as described
above) depends only on a subset of the training data, because the cost
function for building the model does not care about training points
that lie beyond the margin. Analogously, the model produced by Support
Vector Regression depends only on a subset of the training data,
because the cost function for building the model ignores any training
data close to the model prediction.

There are two flavors of Support Vector Regression: :class:`SVR` and
:class:`NuSVR`.

As with classification classes, the fit method will take as
argument vectors X, y, only that in this case y is expected to have
floating point values instead of integer values::

    >>> from sklearn import svm
    >>> X = [[0, 0], [2, 2]]
    >>> y = [0.5, 2.5]
    >>> clf = svm.SVR()
    >>> clf.fit(X, y) # doctest: +NORMALIZE_WHITESPACE
    SVR(C=None, cache_size=200, coef0=0.0, degree=3,
    epsilon=0.1, gamma=0.5, kernel='rbf', probability=False, scale_C=True,
    shrinking=True, tol=0.001, verbose=False)
    >>> clf.predict([[1, 1]])
    array([ 1.5])


.. topic:: Examples:

 * :ref:`example_svm_plot_svm_regression.py`

.. _svm_outlier_detection:

Density estimation, novelty detection
=======================================

One-class SVM is used for novelty detection, that is, given a set of
samples, it will detect the soft boundary of that set so as to
classify new points as belonging to that set or not. The class that
implements this is called :class:`OneClassSVM`.

In this case, as it is a type of unsupervised learning, the fit method
will only take as input an array X, as there are no class labels.

See, section :ref:`outlier_detection` for more details on this usage.

.. figure:: ../auto_examples/svm/images/plot_oneclass_1.png
   :target: ../auto_examples/svm/plot_oneclass.html
   :align: center
   :scale: 75


.. topic:: Examples:

 * :ref:`example_svm_plot_oneclass.py`
 * :ref:`example_applications_plot_species_distribution_modeling.py`


Complexity
==========

Support Vector Machines are powerful tools, but their compute and
storage requirements increase rapidly with the number of training
vectors. The core of an SVM is a quadratic programming problem (QP),
separating support vectors from the rest of the training data. The QP
solver used by this `libsvm`_-based implementation scales between
:math:`O(n_{features} \times n_{samples}^2)` and
:math:`O(n_{features} \times n_{samples}^3)` depending on how efficiently
the `libsvm`_ cache is used in practice (dataset dependent). If the data
is very sparse :math:`n_{features}` should be replaced by the average number
of non-zero features in a sample vector.

Also note that for the linear case, the algorithm used in
:class:`LinearSVC` by the `liblinear`_ implementation is much more
efficient than its `libsvm`_-based :class:`SVC` counterpart and can
scale almost linearly to millions of samples and/or features.


Tips on Practical Use
=====================


  * **Avoiding data copy**: For SVC, SVR, NuSVC and NuSVR, if the data
    passed to certain methods is not C-ordered contiguous, and double
    precision, it will be copied before calling the underlying C
    implementation. You can check whether a give numpy array is
    C-contiguous by inspecting its `flags` attribute.

    For LinearSVC (and LogisticRegression) any input passed as a
    numpy array will be copied and converted to the liblinear
    internal sparse data representation (double precision floats
    and int32 indices of non-zero components). If you want to fit
    a large-scale linear classifier without copying a dense numpy
    C-contiguous double precision array as input we suggest to use
    the SGDClassifier class instead. The objective function can be
    configured to be almost the same as the LinearSVC model.

  * **Kernel cache size**: For SVC, SVR, nuSVC and NuSVR, the size of
    the kernel cache has a strong impact on run times for larger
    problems.  If you have enough RAM available, it is recommended to
    set `cache_size` to a higher value than the default of 200(MB),
    such as 500(MB) or 1000(MB).

  * **Setting C**: In constrast to the scaling in LibSVM and LibLinear,
    the ``C`` parameter in `sklearn.svm` is a per sample penalty.
    Commonly good values for ``C`` often are very large (i.e. ``10**4``)
    and seldom below ``1``.

  * Support Vector Machine algorithms are not scale invariant, so **it
    is highly recommended to scale your data**. For example, scale each
    attribute on the input vector X to [0,1] or [-1,+1], or standardize it
    to have mean 0 and variance 1. Note that the *same* scaling must be
    applied to the test vector to obtain meaningful results. See section
    :ref:`preprocessing` for more details on scaling and normalization.

  * Parameter nu in NuSVC/OneClassSVM/NuSVR approximates the fraction
    of training errors and support vectors.

  * In SVC, if data for classification are unbalanced (e.g. many
    positive and few negative), set class_weight='auto' and/or try
    different penalty parameters C.

  * The underlying :class:`LinearSVC` implementation uses a random
    number generator to select features when fitting the model. It is
    thus not uncommon, to have slightly different results for the same
    input data. If that happens, try with a smaller tol parameter.

  * Using L1 penalization as provided by LinearSVC(loss='l2',
    penalty='l1', dual=False) yields a sparse solution, i.e. only a subset of
    feature weights is different from zero and contribute to the decision
    function.  Increasing C yields a more complex model (more feature are
    selected).  The C value that yields a "null" model (all weights equal to
    zero) can be calculated using :func:`l1_min_c`.


.. _svm_kernels:

Kernel functions
================

The *kernel function* can be any of the following:

  * linear: :math:`<x_i, x_j'>`.

  * polynomial: :math:`(\gamma <x, x'> + r)^d`. `d` is specified by
    keyword ``degree``, `r` by ``coef0``.

  * rbf (:math:`exp(-\gamma |x-x'|^2), \gamma > 0`). :math:`\gamma` is
    specified by keyword ``gamma``.

  * sigmoid (:math:`tanh(<x_i,x_j> + r)`), where `r` is specified by
    ``coef0``.

Different kernels are specified by keyword kernel at initialization::

    >>> linear_svc = svm.SVC(kernel='linear')
    >>> linear_svc.kernel
    'linear'
    >>> rbf_svc = svm.SVC(kernel='rbf')
    >>> rbf_svc.kernel
    'rbf'


Custom Kernels
--------------

You can define your own kernels by either giving the kernel as a
python function or by precomputing the Gram matrix.

Classifiers with custom kernels behave the same way as any other
classifiers, except that:

    * Field `support_vectors\_` is now empty, only indices of support
      vectors are stored in `support_`

    * A reference (and not a copy) of the first argument in the fit()
      method is stored for future reference. If that array changes
      between the use of fit() and predict() you will have unexpected
      results.


Using python functions as kernels
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

You can also use your own defined kernels by passing a function to the
keyword `kernel` in the constructor.

Your kernel must take as arguments two matrices and return a third matrix.

The following code defines a linear kernel and creates a classifier
instance that will use that kernel::

    >>> import numpy as np
    >>> from sklearn import svm
    >>> def my_kernel(x, y):
    ...     return np.dot(x, y.T)
    ...
    >>> clf = svm.SVC(kernel=my_kernel)

.. topic:: Examples:

 * :ref:`example_svm_plot_custom_kernel.py`.

Using the Gram matrix
~~~~~~~~~~~~~~~~~~~~~

Set kernel='precomputed' and pass the Gram matrix instead of X in the
fit method. At the moment, the kernel values between `all` training
vectors and the test vectors must be provided.

    >>> import numpy as np
    >>> from sklearn import svm
    >>> X = np.array([[0, 0], [1, 1]])
    >>> y = [0, 1]
    >>> clf = svm.SVC(kernel='precomputed')
    >>> # linear kernel computation
    >>> gram = np.dot(X, X.T)
    >>> clf.fit(gram, y) # doctest: +NORMALIZE_WHITESPACE
    SVC(C=None, cache_size=200, class_weight=None, coef0=0.0, degree=3,
    gamma=0.0, kernel='precomputed', probability=False, scale_C=True,
    shrinking=True, tol=0.001, verbose=False)
    >>> # predict on training examples
    >>> clf.predict(gram)
    array([ 0.,  1.])

.. _svm_mathematical_formulation:

Mathematical formulation
========================

A support vector machine constructs a hyper-plane or set of hyper-planes
in a high or infinite dimensional space, which can be used for
classification, regression or other tasks. Intuitively, a good
separation is achieved by the hyper-plane that has the largest distance
to the nearest training data points of any class (so-called functional
margin), since in general the larger the margin the lower the
generalization error of the classifier.


.. figure:: ../auto_examples/svm/images/plot_separating_hyperplane_1.png
   :align: center
   :scale: 75

SVC
---

Given training vectors :math:`x_i \in R^p`, i=1,..., n, in two
classes, and a vector :math:`y \in R^n` such that :math:`y_i \in {1,
-1}`, SVC solves the following primal problem:


.. math::

    \min_ {w, b, \zeta} \frac{1}{2} w^T w + C \sum_{i=1, n} \zeta_i



    \textrm {subject to } & y_i (w^T \phi (x_i) + b) \geq 1 - \zeta_i,\\
    & \zeta_i \geq 0, i=1, ..., n

Its dual is

.. math::

   \min_{\alpha} \frac{1}{2} \alpha^T Q \alpha - e^T \alpha


   \textrm {subject to } & y^T \alpha = 0\\
   & 0 \leq \alpha_i \leq C, i=1, ..., l

where :math:`e` is the vector of all ones, C > 0 is the upper bound, Q
is an n by n positive semidefinite matrix, :math:`Q_ij \equiv K(x_i,
x_j)` and :math:`\phi (x_i)^T \phi (x)` is the kernel. Here training
vectors are mapped into a higher (maybe infinite) dimensional space by
the function :math:`\phi`.


The decision function is:

.. math:: sgn(\sum_{i=1}^n y_i \alpha_i K(x_i, x) + \rho)

.. note::

    In practice to have :math:`C` independent of the number of samples :math:`n`,
    :math:`C` is scaled by :math:`n` (Replace :math:`C` by :math:`\frac{C}{n}` in the
    equations above). It corresponds to the scale_C parameter which is True
    by default in all estimators since version 0.11.

.. note::

    While SVM models derived from libsvm and liblinear use *C* as regularization
    parameter, most other estimators use *alpha*. The relation between both is
    :math:`C = \frac{1}{alpha}`.

.. TODO multiclass case ?/

This parameters can be accessed through the members `dual_coef\_`
which holds the product :math:`y_i \alpha_i`, `support_vectors\_` which
holds the support vectors, and `intercept\_` which holds the independent
term :math:`-\rho` :

.. topic:: References:

 * `"Automatic Capacity Tuning of Very Large VC-dimension Classifiers"
   <http://citeseerx.ist.psu.edu/viewdoc/summary?doi=10.1.1.17.7215>`_
   I Guyon, B Boser, V Vapnik - Advances in neural information
   processing 1993,


 * `"Support-vector networks"
   <http://www.springerlink.com/content/k238jx04hm87j80g/>`_
   C. Cortes, V. Vapnik, Machine Leaming, 20, 273-297 (1995)



NuSVC
-----

We introduce a new parameter :math:`\nu` which controls the number of
support vectors and training errors. The parameter :math:`\nu \in (0,
1]` is an upper bound on the fraction of training errors and a lower
bound of the fraction of support vectors.

It can be shown that the `\nu`-SVC formulation is a reparametrization
of the `C`-SVC and therefore mathematically equivalent.


Implementation details
======================

Internally, we use `libsvm`_ and `liblinear`_ to handle all
computations. These libraries are wrapped using C and Cython.

.. _`libsvm`: http://www.csie.ntu.edu.tw/~cjlin/libsvm/
.. _`liblinear`: http://www.csie.ntu.edu.tw/~cjlin/liblinear/

.. topic:: References:

  For a description of the implementation and details of the algorithms
  used, please refer to

    - `LIBSVM: a library for Support Vector Machines
      <http://www.csie.ntu.edu.tw/~cjlin/papers/libsvm.pdf>`_

    - `LIBLINEAR -- A Library for Large Linear Classification
      <http://www.csie.ntu.edu.tw/~cjlin/liblinear/>`_


