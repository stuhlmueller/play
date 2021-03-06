---
layout: model
title: Variational Inference for Bayesian Linear Regression
hidden: true
---

We can try to approach inference in Bayesian Linear Regression using mean-field variational inference:

~~~~
///fold:
var f = function(x, y) {
  return 2*x - 3*y;
};

var data = repeat(20, function(){
  var x = uniform(-5, 5);
  var y = uniform(-5, 5);
  return {
    x: x,
    y: y,
    z: f(x, y)
  };
});

var showMultivariateDist = function(dist) {
  var f = function() {
    var s = sample(dist);
    return {
      x: s.data[0],
      y: s.data[1]
    }
  };
  var out = Infer({method: 'rejection', samples: 5000}, f);
  viz.auto(out);
};
///

var model = function() {
  var mPrior = DiagCovGaussian({
    mu: Vector([0, 0]), 
    sigma: Vector([2, 2])
  });
  var mGuide = DiagCovGaussian({
    mu: param(Vector([0, 0])), 
    sigma: param(Vector([2, 2])) 
  });
  var m = sample(mPrior, {guide: mGuide});

  var bPrior = Gaussian({mu: 0, sigma: 2});
  var bGuide = Gaussian({mu: param(0), sigma: param(2)});
  var b = sample(bPrior, {guide: bGuide});

  // var sigmaPrior = Gamma({shape: 1, scale: 1});
  // var sigmaGuide = Gamma({shape: param(1), scale: param(1)});
  // var sigma = sample(sigmaPrior, {guide: sigmaGuide});
  var sigma = 0.5;

  var f = function(x) {
    return T.sumreduce(T.mul(m, x)) + b;
  };

  var score = sum(map(
    function(datum) {
      var params = {
        mu: f(Vector([datum.x, datum.y])), 
        sigma: sigma
      };
      return Gaussian(params).score(datum.z);
    },
    data));
  factor(score);

  return m;
};

var params = Optimize(model, {steps: 2000, method: {adagrad: {stepSize: 0.01}}, estimator: 'ELBO'});
var marginal = SampleGuide(model, {samples: 1000, params: params});

// showMultivariateDist(marginal)

var E_m0 = expectation(marginal, function(x) { return x.data[0]; });
var E_m1 = expectation(marginal, function(x) { return x.data[1]; });

// Expect to see [2, -3]
[E_m0, E_m1]
~~~~

This doesn't seem to work yet.