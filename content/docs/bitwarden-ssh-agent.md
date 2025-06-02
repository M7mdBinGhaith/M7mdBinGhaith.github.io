---
title: "Bitwarden SSH Agent on Fedora Workstation"
date: 2025-05-27
weight: 20
---
# Bitwarden SSH Agent on Fedora Workstation

## Prerequistes 

*  **Bitwarden Desktop Application in our case flatpak version**
* **Bitwarden SSH Agent Enabled**
* **Public Key deployed to the target server**
---
## Steps

### Make Sure the Bitwarden is running and SSH Agent is Active

* Open your Bitwarden desktop application.
* Go to **Settings**.
* Navigate to the **SSH Agent** section.
* Confirm that the **Enable SSH Agent** option is checked.

### Configure Our Environement to use the Agent

{{< hint info >}}
For this step you need to identify the Socket Path this path will vary depending on how you installed the bitwarden client. In this case we are using the <b>Flatpak</b> version.
{{< /hint >}}
### Execute the following in a terminal shell:
```bash
export SSH_AUTH_SOCK=~/.var/app/com.bitwarden.desktop/data/.bitwarden-ssh-agent.sock
```
### Connect via SSH to verify the agent works:
```bash
 ssh your_username@your_server_ip_or_hostname
```
### Set the variable permenantly in bash or zshrc:
1.  Open your shell configuration file (e.g., `nano ~/.bashrc` for Bash, `nano ~/.zshrc` for Zsh)
2.  Add the `export` command to the end of the file.
3.  Save the file and exit the editor.
4.  Apply the changes:

    ```bash
    source ~/.zshrc
    ```

## Now all of our SSH keys are managed by bitwarden!

            