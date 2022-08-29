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
