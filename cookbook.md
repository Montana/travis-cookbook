# Travis Cookbook (Practical ways at approaching Travis CI) 

Firstly it's important to remember your ```.travis.yml``` file can be linted/validated here: http://lint.travis-ci.org/.

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
 
```yaml
language: elixir

elixir:
  - '1.0.5'
  otp_release: '17.4'

  jobs:
    include:
      - elixir: '1.2'
      otp_release: '18.0'
```

Some things noteworthy of Elixir when it comes to Travis. Travis CI deprecated ```sudo: false```.

```bash
hex.publish
```
Added
```bash
HEX_API_KEY
``` 
Also environment variables and, yes flag CI updates (v0.18). Accounting for these changes, and using the ```git clean -f``` command to address the ```git stash``` issue my deploy section ended up being a lot smaller:

```yaml
deploy:
  provider: script
  script: >-
    mix deps.get &&
    mix hex.publish --yes &&
    git clean -f
  on:
    tags: true
  ```
This approach requires that you generate an API key from ```hex``` and then set it as an Travis CI env var ```(HEX_API_KEY)```. The one way I was able to get around this fairly easily (Montana fix here) is to use ```git stash pop```. Essentially what I did was ```git stash pop```, pop  takes a stashed change, removes it from the “stash stack”, and applies it to your current working tree
      
      
# Python 

Python is one of the most popular languages that are used here at Travis. A classic ```.travis.yml``` file for Python would look like this: 

```yaml
dist: xenial

language: python

cache: pip

python:
    - "3.6"
    - "3.7"
    - "3.8"
    - "nightly"

matrix:
    allow_failures:
        - python: "nightly"

install:
    - pip install pipenv --upgrade-strategy=only-if-needed
    - pipenv install --dev

script:
    - bash scripts/test.sh

after_script:
    - bash <(curl -s https://codecov.io/bash)
 ```
 
 # For Python, let's explain each section: 
 
 ```yaml
 dist: xenial
 ```
 
 ```dist``` in the .yml file is where the Ubuntu Release codename is specified.  Please see: http://releases.ubuntu.com/ for a full list. This specifies the base operating system used for the rest of workflow.
 
 ```yaml
 cache: pip
 ```
```Cache``` allows for a Python package versions to be stored between runtime, to speed up sequential builds. Cache can apply to more than just Python packages.

```yaml
python:
    - "3.6"
    - "3.7"
    - "3.8"
    - "nightly"
  ```
    
Python, given the above language specification, is a key for a sequence of Python versions to perform builds against. Generally most CI tools use the latest bug release version for each minor version. The build logs will tell you the specific versions, you can also view these logs manually via running in debug mode. 

```yaml
matrix:
    allow_failures:
        - python: "nightly"
   ```
```Matrix``` allows for modifications in the above build sequence. In this case, the ```allow_failures``` key specifies a reference to the Python sequence above, and has the value of "nightly", meaning that that version is _possibly_ allowed to fail. Depending on what more build instructions you have.

At some point, you'll have a requirements.txt file, in here this is where you'll have your dependencies: 

```yaml
pip install -r requirements.txt
```

To continue with the Python build, you're going to need to run some updates in the .yml file, via: 

```yaml
sudo apt-get update
sudo apt-get install python3-pip
sudo apt-get install python3-pytest
```

Now add an existing project to GitHub, head over to Travis CI and login, and start the build for Python! 

# Using Bash when deploying a Node project (gh-pages)

Some goals you may have: 

Automating ```GH_REF``` value in gpages_build.sh script with the ```TRAVIS_REPO_SLUG```.

In order to make the ```GH_REF``` variable automatic and not have to use process load everytime, Travis CI gives us the ```TRAVIS_REPO_ShLUG``` which you may have seen in languages like React. The SLUG variable, which is basically the username/repo for whatever GitHub repo you're working on. You write it out like this: ```GH_REF="github.com/${TRAVIS_REPO_SLUG}"```

Below is an example script of just what I've explained above: 

```bash
# This script pushes a demo-friendly version of your element and its
# dependencies to gh-pages.

# usage gp Polymer core-item [branch]
# Run in a clean directory passing in a GitHub org and repo name

GH_REF="github.com/${TRAVIS_REPO_SLUG}"

org=`echo ${TRAVIS_REPO_SLUG} | cut -f 1 -d /`
repo=`echo ${TRAVIS_REPO_SLUG} | cut -f 2 -d /`

name="Montana"
email="montana@travis-ci.org"
branch=${3:-"master"} # default to master, when branch isn't specified

mkdir temp && cd temp # make temp dir 

# make folder (same as input, no checking!)
mkdir $repo
git clone "https://${GH_TOKEN}@${GH_REF}" --single-branch # you can theoretically as Montana likes to do, 'git stash pop' here

# switch to gh-pages branch
pushd $repo >/dev/null
git checkout --orphan gh-pages

# remove all content
git rm -rf -q .

# use bower to install runtime deployment
bower cache clean $repo # ensure we're getting the latest from the desired branch.
git show ${branch}:bower.json > bower.json
echo "{
  \"directory\": \"components\"
}
" > .bowerrc
bower install
bower install $org/$repo#$branch
git checkout ${branch} -- demo
rm -rf components/$repo/demo
mv demo components/$repo/

# redirect by default to the component folder
echo "<META http-equiv="refresh" content=\"0;URL=components/$repo/\">" >index.html

git config user.name $name
git config user.email $email

# send it all to github
git add -A .
git commit -am 'Deploy to GitHub Pages'
git push --force --quiet -u "https://${GH_TOKEN}@${GH_REF}" gh-pages > /dev/null 2>&1

popd >/dev/null
```

# NodeJS 

Sample NodeJS .travis.yml config:

```yaml
language: node_js
node_js: 
  - "stable"
cache:
  directories:
    - "node_modules"
 ```
 
Here's the test spec I created for this cookbook:

```node
const expect = require('chai').expect
const server = require('../index');

# Montana want's it to return a string (just for own reference) - Montana

describe('test', () => {
  it('should return a string', () => {
    expect('ci with travis').to.equal('ci with travis');
  });
});
```
In this case now, run all your classic git commands like: 
 
 ```git
 git init
 git add . 
 git commit -m "Travis build" 
 git remote add origin remote repository URL
 git remote -v 
 git push -u origin master
 ```

# Using Services in Travis (Docker) 

What you'll want to do is grab a Docker image based on what language you're going to use in your .travis.yml file. In this exmple I've picked PHP. So first you're gonna want to open up temrinal and run:

```bash
docker run --name travis-montana -dit travisci/ci-php:packer-1494867192 /sbin/init
```

In the above example but you can see I named it ```montana``` but name the container whatever you want. Just remember what you named it whe you are working the terminal. Once the image is installed, start the Docker container: 

```bash
docker exec -it travis-montana bash -l
```
Now switch to the Travis user 

```bash
su - travs
``` 
Now for Docker, remember you have all the CLI tools installed locally: 

```bash
gem install travis
travis
bundle install
bundler add travis
bundler binstubs travis
```
# Setting up your Docker tests locally:

Genereate a new SSH key for GitHub (using your own email of course)

```bash
ssh-keygen -t rsa -b 4096 -C "your-github-email@example.com"
```

Now copy the contents of the key you’ve just created to your GitHub SSH keys. Name the key something that is easy to identify like _Travis Key_. Assuming you saved your key to the default path you can view it using:

```bash
less ~/.ssh/id_rsa.pub
```
Now clone the repo you want to run tests on, change to your builds direrctory

```bash
cd ~/builds
```
Then clone repo, then change over to the directory the cloe made:

```bash
git clone git@github.com:AUTHOR/PROJECT.git
cd PROJECT
```

Now, to the interesting part, let's compile your Travis build script, we use this vis

```bash
travis buld
```
Now we want to write this to a file called travis.ci -- so the fnal command would look something like: 

```bash
travis compile > travis.ci
```
You'll have to make some changes to this file, whatever editor is easiest for you is fine, in this case I'll be using vim:
```bash
vim.travis.ci
```

You'll want to search for a line similar to: 

```git
branch\=\'\NEW_BRANCH'\
 ```
Change this to your branch name, and run your script!

# Using Docker with Travis CI

For this example we will assume you are using ```python3```

```Dockerfile
from python:3.6
CMD ["python", "-c", "print(12345)"]
```

Next up let's build an image and run it! 

```
docker build -t foobar .


Sending build context to Docker daemon  2.048kB
Step 1/2 : from python:3.6
 ---> 3e4c2972dc8d
Step 2/2 : CMD ["python", "-c", "print(12345)"]
 ---> Running in 5e3273c46264
Removing intermediate container 5e3273c46264
 ---> c1d000f3a768
Successfully built c1d000f3a768
Successfully tagged foobar:latest
```

The image is built using ```foobar``` as a tag in which myself, I’ll use next when running it:

```bash
docker run --rm -ti foobar
12345
```

Some explanations of the arguments below:

- ```--rm``` - Automatically removes the container when it exits
- ```--tty``` -t - Allocate a pseudo-TTY also known as a ```pts```
- ```--interactive , -i``` - Keeps ```STDIN``` open even if not attached to the container

Now let's setup the Travis ```.yml``` file! As you'll notice we will be using ```services``` (as Docker is our service in this method). This file as usual should be listed as ```.travis.yml```.

```yaml
sudo: required
language: python
services:
- docker

script:
- docker build -t foobar .
```

Now let's say this Travis file wants to push to Amazon ECR. Keep in mind that ECR has some limits on maximum amount of tags and images. 

As you see it also expects some environmental variables which I’ll provide in ```.travis.yml.``` Below is an updated script:

```bash
#!/bin/bash -e

# the registry should have been created already
# you could just paste a given url from AWS but I'm
# parameterising it to make it more obvious how its constructed
REGISTRY_URL=${AWS_ACCOUNT_ID}.dkr.ecr.${EB_REGION}.amazonaws.com
# this is most likely namespaced repo name like myorg/veryimportantimage
SOURCE_IMAGE="${DOCKER_REPO}"
# using it as there will be 2 versions published
TARGET_IMAGE="${REGISTRY_URL}/${DOCKER_REPO}"
# lets make sure we always have access to latest image
TARGET_IMAGE_LATEST="${TARGET_IMAGE}:latest"
TIMESTAMP=$(date '+%Y%m%d%H%M%S')
# using datetime as part of a version for versioned image
VERSION="${TIMESTAMP}-${TRAVIS_COMMIT}"
# using specific version as well
# it is useful if you want to reference this particular version
# in additional commands like deployment of new Elasticbeanstalk version
TARGET_IMAGE_VERSIONED="${TARGET_IMAGE}:${VERSION}"

# making sure correct region is set
aws configure set default.region ${EB_REGION}

# Push image to ECR
###################

# I'm speculating it obtains temporary access token
# it expects aws access key and secret set
# in environmental vars
$(aws ecr get-login --no-include-email)

# update latest version
docker tag ${SOURCE_IMAGE} ${TARGET_IMAGE_LATEST}
docker push ${TARGET_IMAGE_LATEST}

# push new version
docker tag ${SOURCE_IMAGE} ${TARGET_IMAGE_VERSIONED}
docker push ${TARGET_IMAGE_VERSIONED}
```
In this scenario I pushed 2 tags, ```latest``` and ```versioned```. AWS also has limits on [this](https://docs.aws.amazon.com/AmazonECR/latest/userguide/service-quotas.html).

As you see it also expects some environmental variables which I’ll provide in ```.travis.yml.``` You'll see an updated script assuming above one was named ```docker_push.sh```:

```yaml
sudo: required
language: python
services:
- docker
env:
  global:
  - DOCKER_REPO=myorg/veryimportantimage
  - EB_REGION="eu-west-1"
  - secure: travisEncryptedAWS_ACCOUNT_ID
  - secure: travisEncryptedAWS_ACCESS_KEY_ID
  - secure: travisEncryptedAWS_SECRET_ACCESS_KEY
before_install:
- pip install awscli
- export PATH=$PATH:$HOME/.local/bin
script:
- docker build -t $DOCKER_REPO .
deploy:
  provider: script
  script: bash docker_push.sh
  on:
    branch: master
 ```
    
Now remember to encrypt your ```env vars``` for our AWS keys, there were three of them:

- AWS_ACCOUNT_ID
- AWS_ACCESS_KEY_ID
- AWS_SECRET_ACCESS_KEY

Since you have Travis installed, read this snippet of [information](https://docs.travis-ci.com/user/environment-variables/#encrypting-environment-variables). So we can run: 

```bash
travis encrypt AWS_ACCOUNT_ID=super_secret --add
```

# Rails

You'll obviously want to signup for Travis, be sure to sync your account, open your ```Gemfile``` and add: 
```ruby
gem 'travis' 
```
Run any migrations you may need to make, then: 

```ruby
bundle install
```
In this use case, we are creating the ```.travis.yml``` via:

```bash
travis init
```
You don't in theory need to specify a Ruby version, Travis will look at your ```ruby-```, but you can definitely specify a Ruby version if you chose to:

```yaml
language: ruby
dist: xenial
rvm: 2.5.8
``` 
That tells Travis what language (Ruby), what ```dist``` you'll be using (also can be changed) and ```rvm``` the version of Ruby you'll be using. You can now add ```script``` which is commands you're telling Travis to run when you triggered a build, so it could look like the following: 

```yaml
script:
 — bundle install — jobs=3 — retry=3
 — bundle exec rake db:create
 — bundle exec rake db:migrate
 — bundle exec rspec
 ```
 
 You can then check your ```.travis.yml```'s validity via the linting online application http://lint.travis-ci.org/, or do it manually: 
 
 ```bash
 travis lint
 ``` 
 
 Then commit changes and see your Rails app build! 

