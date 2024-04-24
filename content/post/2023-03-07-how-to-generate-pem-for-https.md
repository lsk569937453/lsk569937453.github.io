---
title: How to generate pem file for https

description: how to generate pem file for https
date: 2023-03-07T07:07:07+01:00

categories:
 - https
 - RSA
 
tags:
 - https
 - RSA  

---

## Generate the RSA KEY
The following command is to genereate the RSA KEY with password.
```
openssl genrsa -des3 -out privkey.pem 2048 
```
The following command is to genereate the RSA KEY without password.
 
```
openssl genrsa -out privkey.pem 2048 
```

## Generate the Certificate Request
```
openssl req -new -key privkey.pem -out cert.csr 
```
The above command is to use the privkey.pem to generate the Certificate Request.You could take the cert.csr to the CA to apply for the new 
certificate.And the CA will give you a new cacert.pem.


If you just do the testing in your testing environment,you could genereate the cacert.pem with yourself using the following command:
```
openssl req -new -x509 -key privkey.pem -out cacert.pem -days 1095 
```

## Work with the Certificate and Private Key
You could use the privkey.pem and cacert.pem to set up a https server.