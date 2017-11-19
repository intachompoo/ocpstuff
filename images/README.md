## How to generate an s2i image chain from a customized RHEL7 base image using native buildconfigs and imagestreams
#### note that because we are linking images streams, any subsequent modifications to the rhel7-custom image will send imagechange triggers to the entire s2i chain
#### this functionality is higly desired by ops folks when maintaining custom images (and their derived s2i builders)

#### # using the 'openshift' project will allow others to use these images
export PROJECT=openshift

#### make sure you have rhel-7-server-rhscl-rpms and rhel-7-server-optional-rpms repos enabled on all openshift build/app nodes

#### # there is also a requirment that all boxes doing builds be connected to upstream RHN and not local repos (even when all necessary channels are available locally).  
#### # work-around this by enabling your local repos but also registering the nodes and attaching a pool at the same time  TODO: add BZ/RFE link 

#### # build the base rhel7 image from a git repo (clone mine or use it directly to get started)
oc new-build https://github.com/nnachefski/ocpstuff.git --context-dir=/images/rhel7-custom --name=rhel7-custom -n $PROJECT

#### # now build the 'core' s2i image
oc new-build https://github.com/sclorg/s2i-base-container.git -i rhel7-custom --context-dir=core --name=s2i-custom-core --strategy=docker -n $PROJECT

#### # work-around until this feature gets implemented https://bugzilla.redhat.com/show_bug.cgi?id=1382938 
oc patch bc s2i-custom-core -p '{"spec":{"strategy":{"dockerStrategy":{"dockerfilePath": "Dockerfile.rhel7"}}}}' -n $PROJECT
oc start-build s2i-custom-core -n openshift

#### # now build the 'base' s2i image
oc new-build https://github.com/sclorg/s2i-base-container.git -i s2i-custom-core --context-dir=base --name=s2i-custom-base --strategy=docker -n $PROJECT

#### # work-around, see above
oc patch bc s2i-custom-base -p '{"spec":{"strategy":{"dockerStrategy":{"dockerfilePath": "Dockerfile.rhel7"}}}}' -n $PROJECT
oc start-build s2i-custom-base -n $PROJECT

### # now lets build our 'builder' images for each runtime

#### # python 3.5
oc new-build https://github.com/sclorg/s2i-python-container.git -i s2i-custom-base --context-dir=3.5 --name=s2i-custom-python35 --strategy=docker -n $PROJECT

#### # work-around, again
oc patch bc s2i-custom-python35 -p '{"spec":{"strategy":{"dockerStrategy":{"dockerfilePath": "Dockerfile.rhel7"}}}}' -n $PROJECT
oc start-build s2i-custom-python35 -n $PROJECT

#### # build whatever else runtimes you need

### # now lets test with a small python/django app

#### # create a test project
oc new-project custom-s2i-test

#### # allow pull from ‘openshift’ to ‘custom-s2i-test’ 
oc policy add-role-to-group system:image-puller system:serviceaccounts:custom-s2i-test -n openshift

#### # and finally, create the app
oc new-app s2i-python35-custom~https://github.com/nnachefski/pydemo.git --name=pydemo

#### # click to the terminal tab and look for the files that you added to the rhel7-custom base image
  