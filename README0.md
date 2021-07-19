Mistakes made
=============

* Editing sshd_config and adding AllowUsers without including the default user (centos), as I had made a mistake when copying the authorized_keys to the flynn user. This locked me out of the main system, this should of been tested on a dev instance before hand. This could be resolved with access to AWS via stopping the instance, detaching the ebs, attaching it to another, then modifying the sshd_config file and commenting out AllowUsers. Before doing this work 90% was complete only needed to finish off some of the ssh task and nginx tasks.

* Got myself caught in a 503 bad gateway (on the instance I created) using epel_release version of nginx instead of using the latest from Nginx. Tried removing the previous installed package but got myself blocked (created an aws instance on my aws account with an elastic ip at 18.169.211.161 use Interview.pem to access running AWS marketplace version of Centos7). Configured the security group the same as the interview instance so it only has access via 22,80,443 and 8080.

* Disabled postgres unix socket and due to tiredness couldn't get myself back out of it.


How would you harden this system?
=================================

* Instead of using a self signed certificate, either create a cert inside Cert manager in AWS or from another provider (letsencrypt) and then including it inside cert manager (so can be used for example with a alb etc and for safe keeping), this should be modified with a short life that is rotated periodically (you could manage it with Hashicorp vault for example).

* Use an RDS instance instead of installing Postgres on the EC2. Gaining the benefit of being able to manage the instance via IAM (along with HA etc).

* Change the postgres user password from the default and store it inside a vault (e.g. hashicorp Vault/KMS).
* Place the system in a private subnet with access only from within another instance on AWS (bastion/jump host). Preferably this should only be accessible from the system accessing the database (e.g. the JVM instance/image).
* The script shouldn’t be able to access the database directly instead it should probably be controlled via a IAM role.

How would you back the system?
==============================

To backup this system there are multiple ways (snapshot etc). However if this system was in production (in a design of an orchestrator (e.g. nomad or K8s) and containers with an RDS backend). The only thing that I would back up is the RDS data itself, with everything else being created with CaC/IaC. As this wouldn’t be easily be able to be updated otherwise. Multiple methods of doing this can be done from using configuration management (Ansible if a requirement that this system is a PET type) or IAC (e.g. terraform) for creating the infrastructure (with deployment pipelines controlling the deployment of changes Jenkins/CircleCI etc).

All code/config should also be stored within source control (Git), with maybe the method of GitOps being used to store this information (one repo for each environment).

What else would maje this system more reliable to live with over the long term?
===============================================================================

Move everything to separate containers (Docker), inside an orchestration engine (Nomad/ECS/K8s) using a deployment manifest to control the CD side (if using k8s). With a secret management engine in place for storing of secrets (KMS/Hahsicorp Vault).

Additionally if Hashicorp vault was being used you could also use this to control ssh access/password rotation/certificate rotation/aws access control (via assume roles).

Also a load balancer should be put in place to replace nginx web server (either a service mesh/ingress controller like istio/nginx ingress controller or a standard AWS alb).
