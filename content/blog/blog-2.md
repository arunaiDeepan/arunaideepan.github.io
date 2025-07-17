---
title: "What I Learned About Image Caching From a YouTube Rabbit Hole: OpenShift vs Spegel"
date: "2025-07-17"
author: "Arunai"
tags: ["Kubernetes", "OpenShift", "Spegel", "SELinux"]
---

*Ever pushed a new Docker image with the same tag, deployed it, and… nothing changed? Yep. Been there. Here's what I found out - and a really cool tool I wish worked on OpenShift.*

---

## So... Why Isn’t My New Code Showing Up?

Let’s set the scene. You’re working on OpenShift, you build a new image with the same tag (`v1.0.0`, `latest`, whatever), push it, redeploy, and your app is still doing the old stuff.

It’s one of those classic “what the hell is going on?” moments.

Turns out - it’s all about **image caching**, and OpenShift is playing by its own rules (for good reasons).

---

## The Real Reason: OpenShift Caches Images (and It’s Not a Bug)

Here’s the deal: OpenShift (like Kubernetes in general) doesn't automatically pull updated images with the same tag unless you tell it to.

### TL;DR:

* **If the image tag is `latest`**: OpenShift sets `imagePullPolicy: Always`
* **Any other tag?**: It defaults to `IfNotPresent`, meaning “don’t pull again if I already have this tag locally.”

So when you push a new image with `v1.0.0`, OpenShift is like: “Cool, I already have `v1.0.0` cached. I’m good.” And your changes? Nowhere in sight.

[OpenShift Docs: Managing container images](https://docs.redhat.com/en/documentation/openshift_container_platform/4.8/html/images/managing-images#image-pull-policy)

---

## Why This Actually Makes Sense (But Still Hurts)

This behavior isn’t dumb - it’s deliberate:

* **Faster pod startup**
* **Reduced registry bandwidth**
* **Lower pull costs**
* **More reliable startup when the registry is flaky**

It’s just not what you expect when you're in the middle of debugging and nothing looks like it's changing.

---

## Then I Found Spegel… on YouTube

So there I was, doing the usual late-night YouTube spiral, when I randomly stumbled across Spegel in this video titled [Microsoft Tried To Steal A Project And Almost Got Away With It...](https://youtu.be/TsejK1D4y5k?feature=shared) - and yeah, it got interesting real fast. A cool open-source project github.com/spegel-org/spegel

It does something clever: **turns your cluster into a peer-to-peer image-sharing network.** Every node becomes a tiny registry, so when one node has the image, others can grab it from there - **no need to go to an external registry** every time.

Here’s what caught my attention:

* **Super fast deployments** (especially for larger images)
* **Cluster-wide image caching**
* **Reduces external registry hits**
* **Works transparently with existing workflows**

---

## Why Spegel Isn't Available in OpenShift

Now here’s the twist.

I got super excited, then realized... **Spegel doesn't work on OpenShift**. And here's why:

### It’s all about container runtimes: `containerd` vs `CRI-O`

* **Spegel depends on `containerd`** to manage the local image cache and registry mirroring.
* **OpenShift uses `CRI-O`** as its default container runtime - chosen for its Kubernetes focus, tight SELinux integration, and security posture.

While `containerd` is a CNCF project with rich features for pluggable registries, `CRI-O` is **much more minimal** and doesn't expose the same image-layer APIs that Spegel hooks into.

[Why OpenShift uses CRI-O](https://docs.openshift.com/container-platform/latest/architecture/architecture-rhcos.html#rhcos-about-virt-extensions_architecture-rhcos)

So, unless OpenShift shifts to `containerd` (which is unlikely), tools like Spegel just aren't compatible out-of-the-box.

---

## OpenShift's More Traditional Approach

OpenShift’s image caching is still pretty robust, just… different. It leans on:

* **Image streams**: OpenShift’s own abstraction for managing image tags and versions
  [OpenShift Docs: Image Streams](https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html/images/managing-image-streams)
* **Immutable tags**: You should really stop reusing the same tag for different images 😅
* **Explicit pull policies**: You can override the default caching by doing this:

```yaml
spec:
  containers:
    - name: myapp
      image: myregistry.com/myapp:v1.0.0
      imagePullPolicy: Always
```

Or better, just use unique tags for every build:

```bash
podman build -t myapp:${GIT_COMMIT} .
podman push myapp:${GIT_COMMIT}
```

---

## Spegel’s Peer-to-Peer Vibes

If you're running vanilla Kubernetes or something like K3s with `containerd`, Spegel is amazing. It gives you:

* **Instant image access** across the cluster
* **Automatic optimization**
* **Registry-free local pulls**
* **Zero-cost bandwidth reuse**

No more waiting for a 1GB image to pull for the 20th time from Docker Hub.

[Spegel GitHub Project](https://github.com/spegel-org/spegel)

---

## Final Thoughts: Two Philosophies, Two Worlds

This whole deep dive showed me how OpenShift and Spegel represent two very different takes on the same problem:

### OpenShift: The “Stable, Predictable” Way

* Relies on tags being **immutable**
* Optimized for **enterprise-grade stability**
* Makes developers responsible for versioning and caching logic

### Spegel: The “Fast and Distributed” Way

* Cluster becomes its own mini-registry
* **Smart and automatic** caching via P2P
* Geared for **speed, resilience, and cost reduction**

---

## TL;DR: What You Can Do on OpenShift

1. **Don’t reuse tags** - Use something like a Git SHA or timestamp
2. **Set `imagePullPolicy: Always`** when you *really* need to force an update
3. **Leverage ImageStreams** if you’re doing tag promotion workflows
4. **Understand your runtime**  -  tools like Spegel won’t work with CRI-O

---

## Would I Use Spegel?

Absolutely, if I were running containerd-based clusters. It’s clever, fast, and fits well in test/staging or self-hosted production.

But for OpenShift? It’s not the right tool. And that’s fine. OpenShift gives me a more secure, enterprise-grade runtime and tools like ImageStreams that, once you get used to them, work pretty well.

Sometimes the trade-offs are worth it.

---
