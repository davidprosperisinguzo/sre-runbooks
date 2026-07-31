# EC2 Disk Exhaustion During npm install — Root Cause and Fix

**Environment:** AWS EC2 (Ubuntu), provisioned via Terraform, used to test-deploy the WanderLust MERN application
**Resource affected:** Root EBS volume (initial size 8GB)
**Impact:** Instance became unresponsive mid-deployment; initially suspected as memory exhaustion

## Symptom

While setting up the instance to test-run WanderLust, `npm install` stopped responding partway through. The terminal session running it just hung, no further output, no completion, no error.

## Diagnosis Path

### 1. Open a second session to investigate live

Rather than killing the stuck session blind, a second SSH session was opened into the same instance to look at what was actually happening while the first one sat frozen:

```bash
ssh <user>@<instance-ip>
```

### 2. Check system resource usage with btop

`btop` was already installed on the instance from earlier setup, so it was the first thing reached for to get a live read on what the instance was doing:

```bash
btop
```

This immediately ruled out the assumption that would normally come first with an unresponsive instance, memory exhaustion. RAM usage was sitting around 30-40%, with roughly 60% still available. Not even close to exhausted.

Disk usage, on the other hand, showed **90%** utilization. `npm install` had been writing (node_modules, package downloads, build artifacts) until it ran the root volume right up against its ceiling, and the instance effectively stalled once there was nowhere left to write.

## Root Cause

The instance's root EBS volume was provisioned at only 8GB, undersized for a MERN app's `npm install` footprint (`node_modules` alone can be substantial, on top of the base OS and any other installed packages). Once the volume filled to ~90%, further writes from `npm install` caused the process, and effectively the session, to stall.

## Fix Applied

The instance was provisioned through its own IaC repo (separate from `wanderlust-infra`, which handles the application deployment side). The EBS volume size was increased there, from 8GB to 20GB, gp3:

```hcl
resource "aws_instance" "main" {
  # ...
  root_block_device {
    volume_size = 20
    volume_type = "gp3"
  }
}
```

```bash
terraform plan
terraform apply
```

Resizing the EBS volume through Terraform only grows the block device, not the filesystem sitting on top of it, so the filesystem still needed to be extended manually to actually make use of the new space:

```bash
lsblk
sudo growpart /dev/xvda 1
sudo resize2fs /dev/xvda1
```

The original stalled `npm install` process was killed rather than trusting whatever partial state it had left behind, and the install was rerun clean:

```bash
df -h    # confirm new space is available first
npm install
```

## Key Lesson

**Check the resource that's actually cheap to verify before assuming the expensive one.** RAM exhaustion is often the first suspect when an instance goes unresponsive, but `btop` gave a direct, immediate answer that ruled it out in seconds and pointed straight at disk instead. One live look at both metrics side by side settled it faster than reasoning about it would have.

**Resizing the EBS volume is only half the fix.** Terraform growing the declared volume size doesn't automatically extend the filesystem on top of it, `growpart` and `resize2fs` are still required afterward before that new space is actually usable.

**Don't trust a process that stalled mid-write.** Once the disk filled and the install hung, the safer move was killing it and starting fresh rather than assuming it could resume cleanly from wherever it stopped.

## Verifying the Fix Is Complete

```bash
df -h
```

Confirms the volume shows the full 20GB and utilization has dropped to a healthy margin. Then rerunning `npm install` end-to-end without it stalling again is the real confirmation, not just the disk number looking better.

## General Troubleshooting Habit

When an instance goes unresponsive, pull up a live resource monitor (`btop`, `htop`, or similar) before assuming which resource is the bottleneck. It takes seconds to check and rules causes in or out directly, rather than debugging based on which failure mode seems most familiar.
