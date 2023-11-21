FROM python:3

WORKDIR /app

COPY python-app /app

RUN pip install -r requirements

EXPOSE 8888

CMD [ "python",  "/app/app.py" ]
