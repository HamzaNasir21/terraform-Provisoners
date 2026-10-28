# Terraform Provisioners: Deploying a Flask App on AWS EC2

This is a small hands-on project I built while learning Terraform. The goal was to understand **provisioners**, the Terraform feature that lets you run things on a server right after it is created.

With a single `terraform apply`, this project:

1. Builds a small network on AWS (VPC, subnet, internet gateway, route table, security group)
2. Launches an Ubuntu EC2 instance inside it
3. Copies my Flask app (`app.py`) onto the server using the `file` provisioner
4. Logs in over SSH, installs Python, Flask and starts the app using the `remote-exec` provisioner

When it finishes, opening the server's public IP in a browser shows **"Hello, Terraform!"**.

---

## How it fits together

```
                    Internet
                       |
                +------+------+
                |   Internet  |
                |   Gateway   |
                +------+------+
                       |
        VPC 10.0.0.0/16 (eu-north-1)
        +--------------+---------------+
        |   Route table: 0.0.0.0/0     |
        |   -> Internet Gateway        |
        |                              |
        |   Public subnet 10.0.0.0/24  |
        |   +----------------------+   |
        |   |  EC2 (Ubuntu)        |   |
        |   |  Security group:     |   |
        |   |  ports 22 and 80     |   |
        |   |  Flask app on :80    |   |
        |   +----------------------+   |
        +------------------------------+
```

## Project structure

```
.
├── main.tf            # All the infrastructure and the provisioners
├── app.py             # The Flask app that gets deployed to the server
├── .gitignore         # Keeps state files, keys and .terraform/ out of Git
├── README.md
└── screenshots/       # Proof that it works (see the Screenshots section)
```

## Prerequisites

- An AWS account and the AWS CLI configured (`aws configure`)
- Terraform installed
- An SSH key pair on your machine at `~/.ssh/id_rsa` and `~/.ssh/id_rsa.pub`
  (create one with `ssh-keygen -t rsa -b 4096` if you don't have it)

---

## Walkthrough of `main.tf`

Terraform does not run the file top to bottom. It reads every resource, works out which one depends on which (for example, a subnet needs a VPC first) and builds them in the right order. I wrote them in the order below because it is easiest to follow.

### 1. Provider

```hcl
provider "aws" {
  region = "eu-north-1"
}
```

This tells Terraform to talk to AWS and to create everything in the Stockholm region.

### 2. Variable

```hcl
variable "cidr" {
  default = "10.0.0.0/16"
}
```

The IP range of the network. I made it a variable so the VPC does not have the number hardcoded inside it.

### 3. Key pair

```hcl
resource "aws_key_pair" "example" {
  key_name   = "terraform-demo-key"
  public_key = file("~/.ssh/id_rsa.pub")
}
```

This uploads my **public** key to AWS. AWS places it on the server, so I can log in with my private key later. Terraform needs this same key to connect and run the provisioners. The private key never leaves my computer and is not part of this repo.

### 4. VPC

```hcl
resource "aws_vpc" "myvpc" {
  cidr_block = var.cidr
}
```

My own private network inside AWS. Everything else lives inside it.

### 5. Subnet

```hcl
resource "aws_subnet" "sub1" {
  vpc_id                  = aws_vpc.myvpc.id
  cidr_block              = "10.0.0.0/24"
  availability_zone       = "eu-north-1a"
  map_public_ip_on_launch = true
}
```

A slice of the VPC (256 addresses) placed in one availability zone. `map_public_ip_on_launch = true` means any server started here automatically gets a public IP, which is how I reach it from outside.

### 6. Internet gateway

```hcl
resource "aws_internet_gateway" "igw" {
  vpc_id = aws_vpc.myvpc.id
}
```

The door between the VPC and the internet. Without it, the server would be cut off from the outside world.

### 7. Route table

```hcl
resource "aws_route_table" "RT" {
  vpc_id = aws_vpc.myvpc.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.igw.id
  }
}
```

The rule book for traffic. This one says: anything that is not for the local network (`0.0.0.0/0`) should go out through the internet gateway.

### 8. Route table association

```hcl
resource "aws_route_table_association" "rta1" {
  subnet_id      = aws_subnet.sub1.id
  route_table_id = aws_route_table.RT.id
}
```

Attaches the route table to the subnet. Creating a route table alone does nothing until it is linked to a subnet.

### 9. Security group

```hcl
resource "aws_security_group" "webSg" {
  name   = "web"
  vpc_id = aws_vpc.myvpc.id
  ...
}
```

The firewall of the server:

| Direction | Port | Purpose |
|-----------|------|---------|
| Inbound   | 80   | Web traffic, so I can open the Flask app in a browser |
| Inbound   | 22   | SSH, so Terraform and I can log in |
| Outbound  | all  | Lets the server download packages (apt, pip) |

### 10. EC2 instance and the provisioners

This is the main part of the project.

```hcl
resource "aws_instance" "server" {
  ami                    = "ami-0aba19e56f3eaec05"
  instance_type          = "t3.micro"
  key_name               = aws_key_pair.example.key_name
  vpc_security_group_ids = [aws_security_group.webSg.id]
  subnet_id              = aws_subnet.sub1.id
  ...
}
```

This creates a `t3.micro` Ubuntu server inside my subnet, with my key pair and the security group attached. The AMI ID is specific to the `eu-north-1` region.

**The `connection` block** tells Terraform how to reach the server for the provisioners:

```hcl
connection {
  type        = "ssh"
  user        = "ubuntu"
  private_key = file("~/.ssh/id_rsa")
  host        = self.public_ip
}
```

`self.public_ip` means "the public IP of the instance being created right now".

**Provisioner 1: `file`** copies a file from my laptop to the server:

```hcl
provisioner "file" {
  source      = "app.py"
  destination = "/home/ubuntu/app.py"
}
```

**Provisioner 2: `remote-exec`** runs commands on the server:

```hcl
provisioner "remote-exec" {
  inline = [
    "echo 'Hello from the remote instance'",
    "sudo apt update -y",
    "sudo apt-get install -y python3-pip",
    "cd /home/ubuntu",
    "sudo pip3 install flask",
    "sudo python3 app.py &",
  ]
}
```

What each line does:

| Command | What it does |
|---------|--------------|
| `echo ...` | Prints a message, a quick check that the SSH connection works |
| `sudo apt update -y` | Refreshes the package list |
| `sudo apt-get install -y python3-pip` | Installs pip |
| `cd /home/ubuntu` | Moves to the folder where `app.py` was copied |
| `sudo pip3 install flask` | Installs Flask |
| `sudo python3 app.py &` | Starts the app in the background (`sudo` is needed because port 80 is a privileged port) |

Provisioners run in the order they are written, and only **after** the instance has been created.

---

## Walkthrough of `app.py`

```python
from flask import Flask

app = Flask(__name__)

@app.route("/")
def hello():
    return "Hello, Terraform!"

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=80)
```

- `Flask(__name__)` creates the web application.
- `@app.route("/")` says: when someone visits the home page (`/`), run the function below it.
- `hello()` returns the text **Hello, Terraform!**, which the browser displays.
- `host="0.0.0.0"` makes the app listen on all network interfaces. If it only listened on `127.0.0.1`, it would work inside the server but nobody outside could reach it.
- `port=80` is the standard HTTP port, so the app opens at `http://<public-ip>` without typing a port number.

---

## How to run it

```bash
git clone https://github.com/HamzaNasir21/<your-repo-name>.git
cd <your-repo-name>

terraform init      # downloads the AWS provider
terraform plan      # shows what will be created
terraform apply     # builds everything (type yes when asked)
```

When `apply` finishes, copy the instance's public IP from the EC2 console and open it in a browser:

```
http://<public-ip>
```

To log into the server:

```bash
ssh -i ~/.ssh/id_rsa ubuntu@<public-ip>
```

### Clean up

```bash
terraform destroy
```

Always destroy when you are done so AWS does not keep charging for the resources.

---

## Screenshots

**1. Creating the infrastructure with `terraform apply`**

![terraform apply](screenshots/01-terraform-apply.png)

**2. Logging into the EC2 instance over SSH**

![ssh into ec2](screenshots/02-ssh-into-ec2.png)

**3. The Flask app running in the browser**

![app in browser](screenshots/03-app-in-browser.png)

---

## Troubleshooting: the `externally-managed-environment` error

This was the one real problem I hit while building this project.

### What happened

My first `terraform apply` built the network and the server fine, but the `remote-exec` provisioner failed when it reached the Flask install step:

```
error: externally-managed-environment
```

Because a provisioner failed, Terraform marked the instance as **tainted**, which means it would be destroyed and recreated on the next `apply`. So every attempt to test a fix meant waiting for a brand new server.

### Why it happened

The error comes from **PEP 668**. Newer Ubuntu releases mark the system Python as "externally managed", which means `pip` is not allowed to install packages straight into it. The reason is safety: Ubuntu's own tools depend on the system Python, and a global `pip install` could overwrite or break the packages they rely on.

My original command was:

```
sudo pip3 install flask
```

Adding `sudo` does not get around this. The restriction applies to the system-wide Python itself, so it blocks root too. The lesson for me was that this is a policy of the operating system, not a permissions problem.

### How I fixed it

Instead of installing Flask globally, I created a **virtual environment** (`venv`) inside the project folder on the server, installed Flask into it, and started the app with the Python binary from that environment. The virtual environment is its own isolated copy of Python packages, so the system Python is never touched and PEP 668 has nothing to block.

The idea in `remote-exec` looks like this:

```hcl
provisioner "remote-exec" {
  inline = [
    "sudo apt-get update -y",
    "sudo apt-get install -y python3-venv",
    "cd /home/ubuntu",
    "python3 -m venv venv",
    "./venv/bin/pip install flask",
    "sudo ./venv/bin/python app.py &",
  ]
}
```

Two details worth knowing:

- Ubuntu does not always ship the `venv` module by default, so `python3-venv` has to be installed first.
- Flask is installed **inside the venv**, so the app has to be started with `./venv/bin/python`. If I started it with the system `python3`, it would not find Flask. Port 80 is a privileged port, so the final command still needs `sudo`; the venv's Python works fine under `sudo` because it is called by its full path.

### Tip for debugging provisioners

When a `remote-exec` step fails, SSH into the server (`ssh -i ~/.ssh/id_rsa ubuntu@<public-ip>`) and run the failing command by hand. You see the full error immediately, which is much faster than rerunning `terraform apply` every time.

---

## Limitations and what I would improve

I built this to learn how provisioners work, so a few things are intentionally simple. In a real project I would change these:

- **SSH is open to the whole internet** (`0.0.0.0/0` on port 22). I would restrict it to my own IP, for example `x.x.x.x/32`.
- **Provisioners are a last resort.** HashiCorp itself recommends `user_data` / cloud-init, pre-built images (Packer) or a configuration tool like Ansible for setting up servers.
- **The app dies with the server.** `python3 app.py &` is not a managed service, so it will not restart after a reboot or crash. A `systemd` service would fix that.
- **Flask's built-in server is for development.** For production I would run it behind Gunicorn and Nginx.
- **Provisioners only run when the resource is created.** If `remote-exec` fails, Terraform marks the instance as tainted and recreates it on the next apply.
- **The AMI ID and region are hardcoded.** A `data "aws_ami"` lookup would make it portable.
- **I used my default SSH key.** A dedicated key pair for the project would be cleaner.
- **No outputs.** An `outputs.tf` that prints the public IP would save a trip to the console.
