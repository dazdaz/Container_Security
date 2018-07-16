* https://www.katacoda.com/courses/docker-security/image-scanning-with-clair

```
curl -OL https://raw.githubusercontent.com/coreos/clair/master/contrib/compose/docker-compose.yml
# Download Clair's Docker Compose File and Config
mkdir clair_config && curl -L https://raw.githubusercontent.com/coreos/clair/master/config.yaml.sample -o clair_config/config.yaml
# Update config
sed 's/clair-git:latest/clair:v2.0.1/' -i docker-compose.yml && \
sed 's/host=localhost/host=postgres password=password/' -i clair_config/config.yaml
# Start DB
docker-compose up -d postgres
# Populate DB - Download and load the CVE details for Clair to use.
curl -LO https://gist.githubusercontent.com/BenHall/34ae4e6129d81f871e353c63b6a869a7/raw/5818fba954b0b00352d07771fabab6b9daba5510/clair.sql
docker run -it \
    -v $(pwd):/sql/ \
    --network "${USER}_default" \
    --link clair_postgres:clair_postgres \
    postgres:latest \
        bash -c "PGPASSWORD=password psql -h clair_postgres -U postgres < /sql/clair.sql"
# Deploy clair
docker-compose up -d clair

# Scan Image
# Clair works by accepting Image Layers via a HTTP API. To scan all the layers, we need an way to send each layer and aggregate the respond.
# Klar is a simple tool to analyze images stored in a private or public Docker registry for security vulnerabilities using Clair.
curl -L https://github.com/optiopay/klar/releases/download/v1.5/klar-1.5-linux-amd64 -o /usr/local/bin/klar && chmod +x $_

# Using klar, we can now point it at images and see what vulnerabilities they contain, for example quay.io/coreos/clair:v2.0.1.
# The output can be tweaked, for example, only reporting High or Critical vulnerabilities. In this case, we return Low and Above.
CLAIR_ADDR=http://localhost:6060 CLAIR_OUTPUT=Low CLAIR_THRESHOLD=10 klar quay.io/coreos/clair:v2.0.1

# To help with automation, Klar can return the results as JSON. This allows developers plug the scanning into an automated process, such as CI/CD pipeline.
pacman -Syy jq

# By setting the _JSONOUTPUT=true parameter, the results will be in JSON which can be piped to another process, like jq.
CLAIR_ADDR=http://localhost:6060 CLAIR_OUTPUT=High CLAIR_THRESHOLD=10 JSON_OUTPUT=true klar postgres:latest | jq

# Scan a Docker Image held within a private repository
docker run -d --name registry -p 5000:5000 registry:2

# This will be made available via the Katacoda Proxy. Tag an existing Docker Image with the newly started Registry URL.
docker tag postgres:latest 2886795333-5000-cykoria03.environments.katacoda.com/postgres:latest

# Once tagged, the Image can be pushed to the Registry.
docker push 2886795333-5000-cykoria03.environments.katacoda.com/postgres:latest

# In the same way, the private registry image can now be scanned.
CLAIR_ADDR=http://localhost:6060 \
  CLAIR_OUTPUT=Low CLAIR_THRESHOLD=10 \
  klar 2886795333-5000-cykoria03.environments.katacoda.com/postgres:latest
```
