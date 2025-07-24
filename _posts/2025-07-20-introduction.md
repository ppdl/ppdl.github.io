---
title: 00. introduction
author: ppdl
date: 2025-05-20 22:04:35
categories: [systemd]
mermaid: true
tags: [systemd]     # TAG names should always be lowercase
---

## Q1

<div style="display: flex; gap: 1rem; margin-top: 0.5rem; margin-bottom: 0.5rem">
  <div style="flex: 1;">
    <strong>a.service</strong>
    <pre style="background-color: #f0f0f0; padding: 1rem; border-radius: 0.5rem; margin-top: 0.5rem;">
<code class="language-ini">
[Unit]
Description=Service A
DefaultDependencies=no
After=b.service

[Service]
ExecStart=/bin/true
</code>
    </pre>
  </div>

  <div style="flex: 1;">
    <strong>b.service</strong>
    <pre style="background-color: #f0f0f0; padding: 1rem; border-radius: 0.5rem; margin-top: 0.5rem;">
<code class="language-ini">
[Unit]
Description=Service B
DefaultDependencies=no

[Service]
ExecStart=/bin/true

</code>
    </pre>
  </div>
</div>

- systemctl start a.service <span style="color: white;">: A</span>
- systemctl start b.service <span style="color: white;">: B</span>

## Q2

<div style="display: flex; gap: 1rem; margin-top: 0.5rem; margin-bottom: 0.5rem">
  <div style="flex: 1;">
    <strong>a.service</strong>
    <pre style="background-color: #f0f0f0; padding: 1rem; border-radius: 0.5rem; margin-top: 0.5rem;">
<code class="language-ini">
[Unit]
Description=Service A
DefaultDependencies=no
Requires=b.service

[Service]
ExecStart=/bin/true
</code>
    </pre>
  </div>

  <div style="flex: 1;">
    <strong>b.service</strong>
    <pre style="background-color: #f0f0f0; padding: 1rem; border-radius: 0.5rem; margin-top: 0.5rem;">
<code class="language-ini">
[Unit]
Description=Service B
DefaultDependencies=no

[Service]
ExecStart=/bin/true

</code>
    </pre>
  </div>
</div>

- systemctl start a.service <span style="color: white;">: A->B</span>
- systemctl start b.service <span style="color: white;">: B</span>

## Q3

<div style="display: flex; gap: 1rem; margin-top: 0.5rem; margin-bottom: 0.5rem">
  <div style="flex: 1;">
    <strong>a.service</strong>
    <pre style="background-color: #f0f0f0; padding: 1rem; border-radius: 0.5rem; margin-top: 0.5rem;">
<code class="language-ini">
[Unit]
Description=Service A
DefaultDependencies=no
Requires=b.service
After=b.service

[Service]
ExecStart=/bin/true
</code>
    </pre>
  </div>

  <div style="flex: 1;">
    <strong>b.service</strong>
    <pre style="background-color: #f0f0f0; padding: 1rem; border-radius: 0.5rem; margin-top: 0.5rem;">
<code class="language-ini">
[Unit]
Description=Service B
DefaultDependencies=no

[Service]
ExecStart=/bin/true


</code>
    </pre>
  </div>
</div>

- systemctl start a.service <span style="color: white;">: B->A</span>
- systemctl start b.service <span style="color: white;">: B</span>

## Q4

<div style="display: flex; gap: 1rem; margin-top: 0.5rem; margin-bottom: 0.5rem">
  <div style="flex: 1;">
    <strong>a.service</strong>
    <pre style="background-color: #f0f0f0; padding: 1rem; border-radius: 0.5rem; margin-top: 0.5rem;">
<code class="language-ini">
[Unit]
Description=Service A
DefaultDependencies=no
Requires=b.service
Before=b.service

[Service]
ExecStart=/bin/true
</code>
    </pre>
  </div>

  <div style="flex: 1;">
    <strong>b.service</strong>
    <pre style="background-color: #f0f0f0; padding: 1rem; border-radius: 0.5rem; margin-top: 0.5rem;">
<code class="language-ini">
[Unit]
Description=Service B
DefaultDependencies=no

[Service]
ExecStart=/bin/true


</code>
    </pre>
  </div>
</div>

- systemctl start a.service <span style="color: white;">: A->B</span>
- systemctl start b.service <span style="color: white;">: B</span>

## Q5

<div style="display: flex; gap: 1rem; margin-top: 0.5rem; margin-bottom: 0.5rem">
  <div style="flex: 1;">
    <strong>a.service</strong>
    <pre style="background-color: #f0f0f0; padding: 1rem; border-radius: 0.5rem; margin-top: 0.5rem;">
<code class="language-ini">
[Unit]
Description=Service A
DefaultDependencies=no
Requires=b.service

[Service]
ExecStart=/bin/true
</code>
    </pre>
  </div>

  <div style="flex: 1;">
    <strong>b.service</strong>
    <pre style="background-color: #f0f0f0; padding: 1rem; border-radius: 0.5rem; margin-top: 0.5rem;">
<code class="language-ini">
[Unit]
Description=Service B
DefaultDependencies=no
Before=a.service

[Service]
ExecStart=/bin/true
</code>
    </pre>
  </div>
</div>

- systemctl start a.service <span style="color: white;">: B->A</span>
- systemctl start b.service <span style="color: white;">: A</span>

## Q6

<div style="display: flex; gap: 1rem; margin-top: 0.5rem; margin-bottom: 0.5rem">
  <div style="flex: 1;">
    <strong>a.service</strong>
    <pre style="background-color: #f0f0f0; padding: 1rem; border-radius: 0.5rem; margin-top: 0.5rem;">
<code class="language-ini">
[Unit]
Description=Service A
DefaultDependencies=no
Requires=b.service
After=b.service

[Service]
ExecStart=/bin/true
</code>
    </pre>
  </div>

  <div style="flex: 1;">
    <strong>b.service</strong>
    <pre style="background-color: #f0f0f0; padding: 1rem; border-radius: 0.5rem; margin-top: 0.5rem;">
<code class="language-ini">
[Unit]
Description=Service B
DefaultDependencies=no
After=a.service

[Service]
ExecStart=/bin/true

</code>
    </pre>
  </div>
</div>

- systemctl start a.service <span style="color: white;">: Failed to start a.service: Transaction order is cyclic.</span>
- systemctl start b.service <span style="color: white;">: B</span>

## Q7

<div style="display: flex; gap: 1rem; margin-top: 0.5rem; margin-bottom: 0.5rem">
  <div style="flex: 1;">
    <strong>a.service</strong>
    <pre style="background-color: #f0f0f0; padding: 1rem; border-radius: 0.5rem; margin-top: 0.5rem;">
<code class="language-ini">
[Unit]
Description=Service A
DefaultDependencies=no
Requires=b.service

[Service]
ExecStart=/bin/true
</code>
    </pre>
  </div>

  <div style="flex: 1;">
    <strong>b.service</strong>
    <pre style="background-color: #f0f0f0; padding: 1rem; border-radius: 0.5rem; margin-top: 0.5rem;">
<code class="language-ini">
[Unit]
Description=Service B
DefaultDependencies=no
Requires=a.service

[Service]
ExecStart=/bin/true
</code>
    </pre>
  </div>
</div>

- systemctl start a.service <span style="color: white;">: A->B</span>
- systemctl start b.service <span style="color: white;">: A->B</span>

## Q8

<div style="display: flex; gap: 1rem; margin-top: 0.5rem; margin-bottom: 0.5rem">
  <div style="flex: 1;">
    <strong>a.service</strong>
    <pre style="background-color: #f0f0f0; padding: 1rem; border-radius: 0.5rem; margin-top: 0.5rem;">
<code class="language-ini">
[Unit]
Description=Service A
DefaultDependencies=no
After=b.service

[Service]
ExecStart=/bin/true
</code>
    </pre>
  </div>

  <div style="flex: 1;">
    <strong>b.service</strong>
    <pre style="background-color: #f0f0f0; padding: 1rem; border-radius: 0.5rem; margin-top: 0.5rem;">
<code class="language-ini">
[Unit]
Description=Service B
DefaultDependencies=no
After=a.service

[Service]
ExecStart=/bin/true
</code>
    </pre>
  </div>
</div>

- systemctl start a.service <span style="color: white;">: A</span>
- systemctl start b.service <span style="color: white;">: B</span>
