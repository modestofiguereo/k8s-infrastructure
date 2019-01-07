# Installing tools

## Installing AWS CLI

If you already have AWS CLI you can skip this section.

### First let's get PIP
**NOTE:** Depending on your env maybe you should replace `.profile` with `.bash_profile`, or `.bash_login`.
```
curl -O https://bootstrap.pypa.io/get-pip.py
python get-pip.py --user
echo "export PATH=~/.local/bin:$PATH" | tee -a ~/.profile
source ~/.profile
pip --version
```

### Now we can proceed
```
pip install awscli --upgrade --user
aws --version
```

## Installing Kops

Kops is a tool that automates the creation and maintenance of the kubernentes cluster.
```
curl -Lo kops https://github.com/kubernetes/kops/releases/download/$(curl -s https://api.github.com/repos/kubernetes/kops/releases/latest | grep tag_name | cut -d '"' -f 4)/kops-linux-amd64
chmod +x ./kops
sudo mv ./kops /usr/local/bin/
```

## Installing kubectl
We need kubectl in order to interect with the kubernetes cluster:

```
curl -Lo kubectl https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin/kubectl
```
