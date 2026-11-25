# cloud-1
Deploy a website and the necessary docker infrastructure on an instance provided by a cloud provider.

1. ssh-agent (recommended). Before running the playbook, run ssh-add ~/.ssh/cloud-22.pem. The agent holds the key in memory and SSH tries it automatically. The repo has no path at all.
2. Command line. Run ansible-playbook playbook.yml --private-key ~/.ssh/cloud-22.pem. The path is given each time you run it, not stored in the repo.
~/.ssh/config. Add a Host entry with an IdentityFile line. This is per machine and outside the repo.
3. Your own key. Import your usual ~/.ssh/id_ed25519.pub into AWS as a key pair and launch the instances with it. SSH then uses your default key, with no extra setup. This is the cleanest option for a fresh instance at the defense.