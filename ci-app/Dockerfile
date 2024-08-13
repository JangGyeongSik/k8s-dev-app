FROM python:3.6.9

COPY ./ ./

RUN pip3 install -r requirements.txt

WORKDIR /

EXPOSE 80

CMD ["python3", "./app.py"]