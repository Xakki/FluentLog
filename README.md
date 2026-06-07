
# Install with composer
`composer require xakki/fluent-log`

# Php Laravel log (Structured, depended by Monolog)
https://github.com/Xakki/LaraLog

# Php custom log (Only the dependency on the PSR library)
https://github.com/Xakki/PHPErrorCatcher

## Config 

Makefile
```
HOST_NAME ?= $(shell hostname)
# First external IP
HOST_IP  ?= $(shell hostname -I 2>/dev/null | awk '{print $$1}' || echo unknown)
```