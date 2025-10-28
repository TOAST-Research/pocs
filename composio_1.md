# Description

The server code in [Composio master branch](https://github.com/ComposioHQ/composio/blob/47df30bd4de54c9bfc69f07987bc1b549d86c6f3/python/composio/server/api.py#L278) exposes a GET endpoint at /api/download that is intended to serve files from the server. The endpoint takes a file query parameter which specifies the path to the desired file.

The Path(file) operation does not sanitize the user-provided file path. An attacker can use path traversal sequences (e.g., ../) or an absolute path to navigate outside of the intended directory and access any file on the server's filesystem that the application has read permissions for.

# Proof of Concept

## victim setup

run the composio server on localhost:8000

<img width="1516" height="174" alt="Server Initialization" src="https://github.com/TOAST-Research/pocs/blob/main/server_start.png?raw=true" />


## attack steps

GET 127.0.0.1:8000/api/download?file=../../../../../../../../../../xxxx/.ssh/id_rsa then the attacker can read the sensitive file on the returned html


<img width="1444" height="592" alt="Arbitrary Read Proof" src="https://github.com/TOAST-Research/pocs/blob/main/result.png?raw=true" />

<img width="1702" height="94" alt="112836" src="https://github.com/TOAST-Research/pocs/blob/main/read_success.png?raw=true" />


# Impact

A malicious user could abuse this vulnerability to read any file on the victim server like SSH Keys, Confidential information, Internal configuration, Sensitive files, etc.

# Occurrences

api.py L278
