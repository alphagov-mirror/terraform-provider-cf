---
layout: "cloudfoundry"
page_title: "Cloud Foundry: cloudfoundry_v3_droplet"
sidebar_current: "docs-cf-resource-droplet"
description: |-
  Provides a Cloud Foundry Droplet resource.
---

# cloudfoundry_droplet

Provides a Cloud Foundry [application](https://docs.cloudfoundry.org/devguide/deploy-apps/deploy-app.html) resource.

## Example Usage

The following example creates a droplet (package of your built/staged application.

```hcl
resource "cloudfoundry_v3_app" "basic" {
	provider              = cloudfoundry-v3
	name                  = "basic-buildpack"
	space_id              = data.cloudfoundry_v3_space.myspace.id
	environment           = {MY_VAR = "1"}
	instances             = 2
	memory_in_mb          = 1024
	disk_in_mb            = 1024
	health_check_type     = "http"
	health_check_endpoint = "/"
}

resource "cloudfoundry_v3_droplet" "basic" {
	provider         = cloudfoundry-v3
	app_id           = cloudfoundry_v3_app.basic.id
	buildpacks       = ["binary_buildpack"]
	environment      = cloudfoundry_v3_app.basic.environment
	command          = cloudfoundry_v3_app.basic.command
	source_code_path = "/path/to/source.zip"
	source_code_hash = filemd5("/path/to/source.zip")
}
```

## Argument Reference

The following arguments are supported:

* `app_id` - (Required) The GUID of the associated Cloud Foundry application
* `stack` - (Optional) The GUID of the stack the application will be deployed to. Use the [`cloudfoundry_stack`](website/docs/d/stack.html.markdown) data resource to lookup the stack GUID to override Cloud Foundry default.
* `buildpacks` - (Optional, list of strings) The buildpacks used to stage the application. There are multiple options to choose from:
   * a Git URL (e.g. https://github.com/cloudfoundry/java-buildpack.git) or a Git URL with a branch or tag (e.g. https://github.com/cloudfoundry/java-buildpack.git#v3.3.0 for v3.3.0 tag)
   * an installed admin buildpack name (e.g. my-buildpack)
   * an empty blank string to use built-in buildpacks (i.e. autodetection)
* `command` - (Optional, String) A custom start command for the application. (this is only used to trigger rebuild/deployment - it should be set to the output attribute from the `cloudfoundry_v3_app` resource.
* `environment` - (Optional, String) A custom build environment for the application. (this is only used to trigger rebuild/deployment - it should be set to the output attribute from the `cloudfoundry_v3_app` resource.
* `source_code_path` - (Required) An uri or path to target a zip file. this can be in the form of unix path (`/my/path.zip`) or url path (`http://zip.com/my.zip`)
* `source_code_hash` - (Optional) Used to trigger updates. Must be set to a base64-encoded SHA256 hash of the path specified. The usual way to set this is `${base64sha256(file("file.zip"))}`,
where "file.zip" is the local filename of the lambda function source archive.
* `docker_image` - (Optional, String) The URL to the docker image with tag e.g registry.example.com:5000/user/repository/tag or docker image name from the public repo e.g. redis:4.0

~> **NOTE:** [terraform-provider-zipper](https://github.com/ArthurHlt/terraform-provider-zipper)
can create zip file from `tar.gz`, `tar.bz2`, `folder location`, `git repo` locally or remotely and provide `source_code_hash`.

Example Usage with zipper:

```hcl
provider "zipper" {
  skip_ssl_validation = false
}

resource "zipper_file" "fixture" {
  source = "https://github.com/orange-cloudfoundry/gobis-server.git#v1.7.3"
  output_path = "path/to/gobis-server.zip"
}

resource "cloudfoundry_v3_app" "gobis-server" {
    name = "gobis-server"
    source_code_path = zipper_file.fixture.output_path
    source_code_hash = zipper_file.fixture.output_sha
    buildpacks = ["go_buildpack"]
}
```

### Application Deployment strategy (Blue-Green deploy)

* `strategy` - (Required) Strategy to use for creating/updating application. Default to `none`.
Following are supported:
  * `none`:
      * Alias: `standard`, `v2`
      * Description: This is the default strategy, it will restage/create/restart app with interruption
  * `blue-green`:
    * Alias: `blue-green-v2`
    * Description: It will restage and create app without interruption and rollback if an error occurred.

### Service bindings

* `service_binding` - (Optional, Array) Service instances to bind to the application.

  - `service_instance` - (Required, String) The service instance GUID.
  - `params` - (Optional, Map) A list of key/value parameters used by the service broker to create the binding. Defaults to empty map.

~> **NOTE:** Modifying this argument will cause the application to be restaged.
~> **NOTE:** Resource only manages service binding previously set by resource.

### Routing

* `routes` - (Optional, Set) The routes to map to the application to control its ingress traffic. Each route mapping is represented by its `route` block which supports fields documented below.

The `route` block supports:
- `route` - (Required, String) The route id. Route can be defined using the `cloudfoundry_route` resource
- ~`port` - (Number) The port of the application to map the tcp route to.~

~> **NOTE:** in the future, the `route` block will support the `port` attribute illustrated above to allow mapping of tcp routes, and listening on custom or multiple ports.
~> **NOTE:** Route mappings can be controlled from either the `cloudfoundry_app.routes` or the `cloudfoundry_routes.target` attributes. Using both syntaxes will cause conflicts and result in unpredictable behavior.
~> **NOTE:** A given route can not currently be mapped to more than one application using the `cloudfoundry_app.routes` syntax. As an alternative, use the `cloudfoundry_route.target` syntax instead in this specific use-case.
~> **NOTE:** Resource only manages route mapping previously set by resource.

#### Example usage:

```hcl
resource "cloudfoundry_app" "java-spring" {
# [...]
 routes = [
    { route = cloudfoundry_route.java-spring.id },
    { route = cloudfoundry_route.java-spring-2.id }
  ]
}
```

### Environment Variables

* `environment` - (Optional, Map) Key/value pairs of custom environment variables to set in your app. Does not include any [system or service variables](http://docs.cloudfoundry.org/devguide/deploy-apps/environment-variable.html#app-system-env).

~> **NOTE:** Modifying this argument will cause the application to be restaged.

### Health Checks

* `health_check_http_endpoint` -(Optional, String) The endpoint for the http health check type. The default is '/'.
* `health_check_type` - (Optional, String) The health check type which can be one of "`port`", "`process`", "`http`" or "`none`". Default is "`port`".
* `health_check_timeout` - (Optional, Number) The timeout in seconds for the health check.

## Attributes Reference

The following attributes are exported along with any defaults for the inputs attributes.

* `id` - The GUID of the application
* `id_bg` - The GUID of the application updated by resource when strategy is blue-green.
This allow change a resource linked to app resource id to be updated when app will be recreated.

## Timeouts

* App instance startup timeout - see the `timeout` argument.
* App staging timeout - 15 mins.
* Service binding timeout - 5 mins.

## Import

The current App can be imported using the `app` GUID, e.g.

```bash
$ terraform import cloudfoundry_app.spring-music a-guid
```

## Update resource using blue green app id

This is an example of usage of `id_bg` attribute to update your resource on a changing app id by blue-green:

```hcl
resource "cloudfoundry_app" "test-app-bg" {
    space            = "apaceid"
    buildpack        = "abuildpack"
    name             = "test-app-bg"
    path             = "myapp.zip"
    strategy         = "blue-green-v2"
    routes {
      route = "arouteid"
    }
}

resource "cloudfoundry_app" "test-app-bg2" {
  space            = "apaceid"
  buildpack        = "abuildpack"
  name             = "test-app-bg2"
  path             = "myapp.zip"
  strategy         = "blue-green-v2"
  routes {
    route = "arouteid"
  }
}

resource "cloudfoundry_network_policy" "my-policy" {
  policy {
    destination_app = cloudfoundry_app.test-app-bg2.id_bg
    port            = "8080"
    protocol        = "tcp"
    source_app      = cloudfoundry_app.test-app-bg.id_bg
  }
}

# When you change either test-app-bg or test-app-bg2 this will affect my-policy to be updated because it use `id_bg` instead of id
```