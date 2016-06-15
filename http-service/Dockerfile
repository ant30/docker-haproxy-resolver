FROM python:3.4.4-wheezy

ADD index.html /srv/

RUN apt-get update && apt-get install -y dnsutils && apt-get clean

WORKDIR /srv

CMD ["python", "-m", "http.server", "9000"]

EXPOSE "9000"
