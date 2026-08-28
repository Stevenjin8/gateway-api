---
title: "GEP-4360: Regex Path Rewrites"
---

* Issue: [#4359](https://github.com/kubernetes-sigs/gateway-api/issues/4359)
* Status: Experimental

## TLDR

Right now Gateway API supports only full path or prefix rewrites, we want to extend it to regex-based path rewrites.
This is already supported by [Envoy](https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/route/v3/route_components.proto#envoy-v3-api-field-config-route-v3-routeaction-regex-rewrite),
[NGINX](https://nginx.org/en/docs/http/ngx_http_rewrite_module.html#rewrite),
and [HAProxy](https://cbonte.github.io/haproxy-dconv/2.5/configuration.html#4.2-http-request%20replace-path);
in this proposal we are closing the gap between Gateway API and current capabilities of the modern LBs.

## Goals

Close the regex-based path rewrites feature gap for Gateway API, i.e.:

 * Rewrite the path of a request based on a regular expression, regardless of initial match type
 * Substitute matching section(s) in the regular expression with predefined values

## Non-Goals

  * Any sort of host rewriting

## Introduction/Overview

We would like to add an enhancement to the HTTPURLRewriteFilter that would allow the caller to specify path rewrite based on the provided pattern and substitution.
Right now Gateway API supports only full path or prefix rewrites, we want to extend it taking into account capabilities of the modern LBs.

## Purpose (Why and Who)

This is a highly requested feature. This is also supported by Envoy, NGINX, and HAProxy.

In this proposal we are closing the gap between Gateway API and current capabilities of the modern LBs.

## Implementation and Support

| Implementation | Support | Engine |
|----------------|------------|----------------|
| Envoy | [config.route.v3.RouteAction.regex_rewrite](https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/route/v3/route_components.proto#envoy-v3-api-field-config-route-v3-routeaction-regex-rewrite) | RE2 |
| HAProxy | [http-request replace-path](https://cbonte.github.io/haproxy-dconv/2.5/configuration.html#4.2-http-request%20replace-path) | PCRE |
| NGINX | [ngx_http_rewrite_module.html#rewrite](https://nginx.org/en/docs/http/ngx_http_rewrite_module.html#rewrite) | PCRE |

NGINX only replaces the first match of the pattern using the rewrite directive, but you can get full substitution using Lua.

## API

This GEP adds the `ReplaceRegularExpression` path modifier type and its corresponding `replaceRegularExpression` field to `HTTPPathModifier`.

```go
// HTTPPathModifierType defines the type of path redirect or rewrite.
type HTTPPathModifierType string

const (
	// ...

	// RegularExpressionHTTPPathModifier indicates that the path will be
	// replaced according to the configured regular expression.
	//
	// See [Gateway API Regex](https://gateway-api.sigs.k8s.io/geps/gep-4359/)
	// for portability requirements. Expressions outside of Gateway API Regex
	// are not portable across implementations.
	//
	// <gateway:experimental>
	RegularExpressionHTTPPathModifier HTTPPathModifierType = "ReplaceRegularExpression"
)

// HTTPPathModifier defines configuration for path modifiers.
type HTTPPathModifier struct {
	// ...

	// ReplaceRegularExpression specifies a replace rule given by a Gateway API Regex expression and a substitution string.
	//
    // See [Gateway API Regex](https://gateway-api.sigs.k8s.io/geps/gep-4359/)
    // for portability requirements. Expressions outside of Gateway API Regex are
    // not portable across implementations.
    //
	// Support: Extended
	//
	// +optional
	// <gateway:experimental>
	ReplaceRegularExpression *HTTPPathRegularExpressionModifier `json:"replaceRegularExpression,omitempty"`
}

// HTTPPathRegularExpressionModifier defines a regular-expression-based path
// replacement.
type HTTPPathRegularExpressionModifier struct {
	// Pattern is a Gateway API Regex expression that is matched against the
	// request path.
	//
    // See [Gateway API Regex](https://gateway-api.sigs.k8s.io/geps/gep-4359/)
    // for portability requirements. Expressions outside of Gateway API Regex are
    // not portable across implementations.
    //
	// +kubebuilder:validation:MaxLength=1024
	// +required
	Pattern string `json:"pattern"`

	// Substitution is the value that replaces each match. Capturing groups are
	// referenced with \1, \2, and so on.
	//
	// +kubebuilder:validation:MaxLength=1024
	// +required
	Substitution string `json:"substitution"`
}
```

For example, the following rewrites `/api/v1/users` to `/v1/users`:

```yaml
filters:
- type: URLRewrite
  urlRewrite:
    path:
      type: ReplaceRegularExpression
      replaceRegularExpression:
        pattern: ^/api/(.*)$
        substitution: /\1
```
