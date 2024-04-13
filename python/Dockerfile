
## Use Small Docker Base Images
FROM python:3.12.3-slim

#Labels as key value pair
LABEL Maintainer="dperez"
# Any working directory can be chosen
WORKDIR /app

## Prefer COPY Over ADD
COPY requirements.txt .

#RUN apt-get update && apt-get install -y \
#    netcat \
#    && apt-get clean \
#    && rm -rf /var/lib/apt/lists/*

RUN pip3 install -r requirements.txt

COPY example.py ./


## Use Unprivileged Containers
RUN addgroup --system app && adduser --system --group app

USER app

EXPOSE 5000

## Prefer Array Over String Syntax
CMD [ "python", "./example.py"]
