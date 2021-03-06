=================
Swift ConfigInfo
=================

A middleware that gives information about the current cluster.

The middleware responds to any request to "/configinfo" with a Response containing a JSON formatted snapshot of the configuration. The admin can specify which parameters are published, during the install inside the configuration file of the proxy server.

Here's the current state of how it operates:

- The middleware parses the conf file it gets from the environment (`conf["__file__"]`) inside a ConfigParser object.
- It will then select the sections according to its own configuration.
- It formats this into a JSON and dumps it in the body of the response.


*******
Install
*******

1. Install the middleware with pip
::
   pip install https://github.com/enovance/swift-config-info/zipball/master


2. Add configinfo in your proxy pipeline. If you want to allow anonymous querying, make sure to include the middleware before any of the auth middlewares.
::

   [pipeline:main]
   pipeline = catch_errors healthcheck proxy-logging configinfo cache bulk slo ratelimit   authtoken keystoneauth tempauth  tempurl formpost staticweb container-quotas account-quotas proxy-logging proxy-server

3. Configure the *configinfo* middleware by specifying the publicly available parameters as a comma-separated list of section names
::

   [filter:configinfo]
   use = egg:swift_config_info#configinfo
   public_conf_sections = pipeline:main, filter:tempurl

*****
Usage
*****

Once the middleware is installed, testing the middleware is as simple as sending a GET request to "http://<swift_address>/configinfo". You can enhance the visual formatting by piping the command into `python -m tools`. For instance:
::
   curl 'http://192.168.33.10:8080/configinfo | python -m json tools

   {
    "filter:tempurl": {
        "bind_port": "8080",
        "log_level": "DEBUG",
        "swift_dir": "/etc/swift",
        "use": "egg:swift#tempurl",
        "user": "vagrant",
        "workers": "1"
    },
    "pipeline:main": {
        "bind_port": "8080",
        "log_level": "DEBUG",
        "pipeline": "catch_errors healthcheck proxy-logging configinfo cache bulk slo ratelimit   authtoken keystoneauth tempauth  tempurl formpost staticweb container-quotas account-quotas proxy-logging proxy-server",
        "swift_dir": "/etc/swift",
        "user": "vagrant",
        "workers": "1"
    }
   }

If one of the sections defined in `public_conf_sections` doesn't exist in the conf file, then *configinfo* will ignore it and return `null` for its value. For instance, looking for a non exsting *badsection* config, here's the returned JSON object:
::
  {
      "badsection": null,
      "filter:tempurl": {
          "bind_port": "8080",
          "log_level": "DEBUG",
          "swift_dir": "/etc/swift",
          "use": "egg:swift#tempurl",
          "user": "vagrant",
          "workers": "1"
      },
      "pipeline:main": {
          "bind_port": "8080",
          "log_level": "DEBUG",
          "pipeline": "catch_errors healthcheck proxy-logging capability cache bulk slo ratelimit   authtoken keystoneauth tempauth  tempurl formpost staticweb container-quotas account-quotas proxy-logging proxy-server",
          "swift_dir": "/etc/swift",
          "user": "vagrant",
          "workers": "1"
      }
  }


It's possibe to query the middleware for a single section, or a single option in the section, by appending `/section(/option)` in the request URL:
::

   curl 'http://192.168.33.10:8080/configinfo/filter:tempurl' | python -m json tools

   {
    "filter:tempurl": {
        "bind_port": "8080",
        "log_level": "DEBUG",
        "swift_dir": "/etc/swift",
        "use": "egg:swift#tempurl",
        "user": "vagrant",
        "workers": "1"}
   }

   curl 'http://192.168.33.10:8080/configinfo/filter:tempurl/log_level'

   "DEBUG"

If the section or the option requested is not available in `public_config`, the middleware returns a 404 HTTPNotFound error.
