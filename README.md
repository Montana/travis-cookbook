## Travis Cookbook 

Firstly it's important to remember your Travis ```.yml``` file can be linted/validated here: http://lint.travis-ci.org/.

# Supported languages 

For supported languages, take a look here: https://docs.travis-ci.com/user/languages/ (note: R is community based). 

# Classic Java .yml file 

Below is a yml file that leverages Gradle, this is a sample of how it would be used in Travis: 

```yaml
language: java
# specify version of Java
jdk:
- oraclejdk8

sudo: false

# cache the build tools, or use container infra to speed up builds (ref Montana's LXD guide) 
cache:
  directories:
  - $HOME/.m2
  - $HOME/.gradle
  ```

With classic Java builds, if there is a Gradle wrapper available Travis CI should build your project via using the ```gradlew build``` command. The very first build must be be triggered by a ```git push```. After that it can be triggered by the ```restart build``` button in the UI. 

The Gradle config (if you're using Gradle) can be configured even more via caching options: 


```yaml
before_cache:
 - rm -f $HOME/.gradle/caches/modules-2/modules-2.lock
cache:
 directories:
 - $HOME/.gradle/caches/
 - $HOME/.gradle/wrapper/
 - $HOME/.gradle/nodejs/
 - node_modules
 ```
 
 # Elixir
 
 Elixir is a little niche language, so I've addeed a video for added support. So let's say you have a .travis.yml build file for Elixir that reads like this:
 
<pre>
language: elixir

elixir:
  - '1.0.5'
  otp_release: '17.4'

  jobs:
    include:
      - elixir: '1.2'
      otp_release: '18.0'
      </pre>
      
This is a pretty basic setup to get your Elixir app in the build pipeline, herer's a video for further details: 

