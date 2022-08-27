```sh
mkdir $HOME/jenkins_home

docker run -d -p 8080:8080 \
  -v $HOME/jenkins_home:/var/jenkins_home \
  --name jenkins jenkins/jenkins

docker logs jenkins

#access http://localhost:8080
```