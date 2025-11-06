# Google Cloud - Zonal Managed Instance Groups

## Step-01: Introduction
In this guide, we will learn:
1. Create Regional Health Check (TCP)
2. Create Firewall Rule for health checks
3. Create Instance Template
4. Create Zonal Managed Instance Groups (ZMIGs)

**Why are we doing this?**  
Zonal Managed Instance Groups (ZMIGs) differ from Regional MIGs in important ways:
- **Zonal scope**: Instances are in a single zone (not distributed across zones automatically)
- **Manual multi-zone**: You create separate ZMIGs in each zone you want coverage
- **More control**: Fine-grained control over instance placement
- **Regional load balancing**: Used with Regional Load Balancers (not Global)
- **Lower latency**: Keep traffic within a region for compliance or performance

**When to use Zonal MIGs vs Regional MIGs:**

| Zonal MIG | Regional MIG |
|-----------|--------------|
| Need precise zone control | Want automatic zone distribution |
| Regional Load Balancer | Global Load Balancer |
| Single-region applications | Multi-region applications |
| Specific zone requirements | Simplified management |
| Cost optimization per zone | High availability priority |

---

## Step-02: Create Regional Health Check - TCP

**What we do:**
```bash
# Set Project
gcloud config set project ania-levelup

# Create Regional Health Check (TCP protocol)
gcloud compute health-checks create tcp regional-tcp-health-check \
    --port=80 \
    --region=us-central1 

# Verify health check creation
gcloud compute health-checks list
gcloud compute health-checks describe regional-tcp-health-check --region=us-central1
```

**Why are we doing this?**  
Regional health checks are different from global health checks:

**Regional Health Check:**
- **Scope**: Only available within a specific region (us-central1)
- **Use case**: Used with Regional Load Balancers and Zonal MIGs
- **Protocol: TCP**: Checks if port 80 is responding (faster than HTTP checks)
- **Port 80**: Standard HTTP port where nginx listens

**How TCP health check works:**
1. Health check system attempts to connect to port 80
2. If connection succeeds → instance is HEALTHY
3. If connection fails → instance is UNHEALTHY
4. Unhealthy instances are automatically replaced by MIG

**TCP vs HTTP health checks:**
- **TCP**: Just checks if port is open (faster, less overhead)
- **HTTP**: Checks for specific HTTP response codes (more thorough)
- For simple web servers, TCP is usually sufficient

**Regional vs Global health checks:**
| Regional | Global |
|----------|--------|
| Single region only | Available everywhere |
| Used with Regional LB | Used with Global LB |
| Lower latency checks | Higher latency checks |
| Cannot be shared across regions | Shared globally |

---

## Step-03: Create Firewall Rules

**What we do:**

**Important Note:** This firewall rule was already created in the Regional MIG tutorial. If it exists, skip this step. If not, create it:

```bash
# Verify if the rule already exists
gcloud compute firewall-rules describe vpc3-custom-allow-health-check

# If it doesn't exist, create it:
# Firewall Rule: Ingress rule that allows traffic from Google Cloud health checking systems
gcloud compute firewall-rules create vpc3-custom-allow-health-check \
    --network=vpc3-custom \
    --description="Allows traffic from Google Cloud health checking systems" \
    --direction=INGRESS \
    --source-ranges=130.211.0.0/22,35.191.0.0/16 \
    --action=ALLOW \
    --rules=tcp:80

# Verify firewall rule
gcloud compute firewall-rules list --filter="network:vpc3-custom"
gcloud compute firewall-rules describe vpc3-custom-allow-health-check
```

**Why are we doing this?**  
This firewall rule is CRITICAL for health checks to work:

**Source ranges explained:**
- **130.211.0.0/22**: Google Cloud health check systems (primary range)
- **35.191.0.0/16**: Google Cloud health check systems (secondary range)

**Why these specific IPs?**
- These are Google's health check probe source addresses
- Health checks originate from these IP ranges
- Without this rule, health checks are blocked
- Blocked health checks = instances marked UNHEALTHY
- UNHEALTHY instances = constantly recreated by MIG

**What happens without this rule:**
1. ❌ Health checks cannot reach instances
2. ❌ All instances marked as UNHEALTHY
3. ❌ MIG tries to replace "unhealthy" instances
4. ❌ New instances also fail health checks
5. ❌ Continuous recreation loop
6. ❌ Application never becomes available

**Critical for MIG operation:**
- MIGs rely on health checks for auto-healing
- Without health checks, auto-healing doesn't work
- This rule must exist before creating MIGs

---

## Step-04: Create Instance Template

**What we do:**

**Important Note:** Ensure nginix-vpc.sh is present in your current directory before running this command.

```bash
# us-central1: Create Instance Template
gcloud compute instance-templates create it-rlbdemo-us-central1 \
    --region=us-central1 \
    --network=vpc3-custom \
    --subnet=us-central1-subnet \
    --tags=lb-tag \
    --machine-type=e2-micro \
    --metadata-from-file=startup-script=nginix-vpc.sh

# Verify instance template creation
gcloud compute instance-templates list
gcloud compute instance-templates describe it-rlbdemo-us-central1
```

**Why are we doing this?**  
Instance templates define the blueprint for all VMs in the MIG:

**Template configuration:**
- **Name**: it-rlbdemo-us-central1 (indicates regional load balancer demo)
- **Region**: us-central1 (regional resource)
- **Network**: vpc3-custom (our custom VPC)
- **Subnet**: us-central1-subnet (10.135.0.0/20 range)
- **Tags**: lb-tag (for firewall rules and identification)
- **Machine type**: e2-micro (cost-effective for demos)
- **Startup script**: nginix-vpc.sh (installs and configures nginx)

**Network tags (lb-tag):**
- Used to apply firewall rules selectively
- Can be used for load balancer targeting
- Helps identify instances created from this template
- Useful for organization and security

**Why use tags:**
1. Apply firewall rules to specific instances
2. Identify instances in load balancer backend
3. Organize resources for billing and monitoring
4. Security segmentation

**Template benefits:**
- ✅ All instances created identically
- ✅ Easy to update (create new template version)
- ✅ Startup script ensures consistent configuration
- ✅ Can create instances in multiple zones from same template

---

## Step-05: Create Zonal Managed Instance Groups

**What we do:**
```bash
# Zone: us-central1-a - Create Zonal Managed Instance Group
gcloud compute instance-groups managed create zmig-us-1 \
    --zone=us-central1-a \
    --size=2 \
    --template=it-rlbdemo-us-central1 

# Verify zmig-us-1 creation
gcloud compute instance-groups managed describe zmig-us-1 --zone=us-central1-a

# Zone: us-central1-c - Create Zonal Managed Instance Group
gcloud compute instance-groups managed create zmig-us-2 \
    --zone=us-central1-c \
    --size=2 \
    --template=it-rlbdemo-us-central1 

# Verify zmig-us-2 creation
gcloud compute instance-groups managed describe zmig-us-2 --zone=us-central1-c

# List all MIGs
gcloud compute instance-groups managed list

# List instances created by both ZMIGs
gcloud compute instances list --filter="name~'zmig-us-.*'"

# Get detailed info about instances in each ZMIG
echo "=== Instances in zmig-us-1 (zone us-central1-a) ==="
gcloud compute instance-groups managed list-instances zmig-us-1 --zone=us-central1-a

echo "=== Instances in zmig-us-2 (zone us-central1-c) ==="
gcloud compute instance-groups managed list-instances zmig-us-2 --zone=us-central1-c
```

**Why are we doing this?**  
We create two separate Zonal MIGs for multi-zone coverage:

**ZMIG 1 (zmig-us-1):**
- **Zone**: us-central1-a
- **Size**: 2 instances
- **Template**: it-rlbdemo-us-central1
- **Purpose**: Provide capacity in zone a

**ZMIG 2 (zmig-us-2):**
- **Zone**: us-central1-c
- **Size**: 2 instances
- **Template**: it-rlbdemo-us-central1
- **Purpose**: Provide capacity in zone c

**Why two separate ZMIGs?**
- **High availability**: If one zone fails, other zone continues
- **Zone isolation**: Each ZMIG is independent
- **Manual control**: You decide which zones to use
- **Load distribution**: Spread load across zones

**Total architecture:**
```
Region: us-central1
├─ Zone: us-central1-a (zmig-us-1)
│  ├─ Instance 1 (10.135.x.x)
│  └─ Instance 2 (10.135.x.x)
└─ Zone: us-central1-c (zmig-us-2)
   ├─ Instance 1 (10.135.x.x)
   └─ Instance 2 (10.135.x.x)

Total: 4 instances across 2 zones
```

**Zonal vs Regional MIG comparison:**

**Zonal MIG (what we just created):**
```bash
# Create separate MIG per zone
gcloud compute instance-groups managed create zmig-us-1 --zone=us-central1-a
gcloud compute instance-groups managed create zmig-us-2 --zone=us-central1-c
# Result: 2 separate MIGs to manage
```

**Regional MIG (from previous tutorial):**
```bash
# Create one MIG, specify multiple zones
gcloud compute instance-groups managed create mig1 \
    --zones=us-central1-a,us-central1-c
# Result: 1 MIG automatically distributes across zones
```

**Key differences:**

| Aspect | Zonal MIG | Regional MIG |
|--------|-----------|--------------|
| **Management** | Manage each ZMIG separately | Single MIG manages all zones |
| **Scaling** | Scale each zone independently | Automatic distribution across zones |
| **Complexity** | More complex (multiple MIGs) | Simpler (one MIG) |
| **Control** | Fine-grained per-zone control | Automatic zone balancing |
| **Use case** | Regional Load Balancer | Global Load Balancer |
| **Flexibility** | Can have different sizes per zone | Balanced across zones |

**When to use Zonal MIGs:**
- ✅ Need different instance counts per zone
- ✅ Using Regional Load Balancer
- ✅ Want precise control over placement
- ✅ Zone-specific configurations
- ✅ Cost optimization per zone

**When to use Regional MIGs:**
- ✅ Want simplified management
- ✅ Using Global Load Balancer
- ✅ Need automatic zone balancing
- ✅ High availability is priority
- ✅ Don't need per-zone customization

---

## Step-06: Attach Health Check to ZMIGs

**What we do:**
```bash
# Attach regional health check to zmig-us-1
gcloud compute instance-groups managed set-autohealing zmig-us-1 \
    --health-check=regional-tcp-health-check \
    --initial-delay=300 \
    --zone=us-central1-a

# Attach regional health check to zmig-us-2
gcloud compute instance-groups managed set-autohealing zmig-us-2 \
    --health-check=regional-tcp-health-check \
    --initial-delay=300 \
    --zone=us-central1-c

# Verify health check attachment
echo "=== Health check status for zmig-us-1 ==="
gcloud compute instance-groups managed describe zmig-us-1 \
    --zone=us-central1-a \
    --format="get(autoHealingPolicies)"

echo "=== Health check status for zmig-us-2 ==="
gcloud compute instance-groups managed describe zmig-us-2 \
    --zone=us-central1-c \
    --format="get(autoHealingPolicies)"
```

**Why are we doing this?**  
Health checks enable auto-healing for the ZMIGs:

**Auto-healing configuration:**
- **Health check**: regional-tcp-health-check
- **Initial delay**: 300 seconds (5 minutes)
- **Purpose**: Automatically replace unhealthy instances

**Initial delay explained:**
- **300 seconds**: Wait 5 minutes before starting health checks
- **Why?**: Gives instances time to boot and start services
- **Important**: Nginx takes time to install via startup script
- **Too short**: Healthy instances marked unhealthy during startup
- **Too long**: Delays detection of actually unhealthy instances

**Auto-healing process:**
1. Instance boots and runs startup script (3-5 minutes)
2. Initial delay passes (5 minutes in our config)
3. Health check starts testing port 80
4. If instance fails health checks → marked UNHEALTHY
5. MIG automatically deletes unhealthy instance
6. MIG creates replacement instance from template
7. New instance goes through same process

**Benefits of auto-healing:**
- ✅ Self-healing infrastructure
- ✅ No manual intervention needed
- ✅ Maintains desired instance count
- ✅ Replaces crashed/frozen instances
- ✅ Improves application reliability

---

## Step-07: Configure Named Ports for ZMIGs

**What we do:**
```bash
# Set named port for zmig-us-1
gcloud compute instance-groups set-named-ports zmig-us-1 \
    --named-ports=webserver80:80 \
    --zone=us-central1-a

# Set named port for zmig-us-2
gcloud compute instance-groups set-named-ports zmig-us-2 \
    --named-ports=webserver80:80 \
    --zone=us-central1-c

# Verify named ports
echo "=== Named ports for zmig-us-1 ==="
gcloud compute instance-groups describe zmig-us-1 \
    --zone=us-central1-a \
    --format="get(namedPorts)"

echo "=== Named ports for zmig-us-2 ==="
gcloud compute instance-groups describe zmig-us-2 \
    --zone=us-central1-c \
    --format="get(namedPorts)"
```

**Why are we doing this?**  
Named ports are required for load balancer integration:

**Named port configuration:**
- **Name**: webserver80
- **Port**: 80 (HTTP)
- **Purpose**: Labels port 80 for load balancer

**Why named ports are needed:**
- Load balancers reference ports by name, not number
- Makes configuration more flexible and readable
- Can change port numbers without updating load balancer
- Required for backend service configuration

**How load balancers use named ports:**
```
Load Balancer Backend Service
    ↓
References "webserver80" named port
    ↓
Translates to port 80
    ↓
Sends traffic to port 80 on instances
```

**Without named ports:**
- ❌ Cannot attach instance group to load balancer
- ❌ Backend service configuration fails
- ❌ No way to route traffic to instances

**With named ports:**
- ✅ Load balancer knows which port to use
- ✅ Can attach to backend service
- ✅ Traffic routes correctly
- ✅ Easy to change port if needed

---

## Step-08: Verify All Resources

**What we do:**
```bash
# 1. Verify VPC and Subnet
gcloud compute networks describe vpc3-custom
gcloud compute networks subnets describe us-central1-subnet --region=us-central1

# 2. Verify Firewall Rules
gcloud compute firewall-rules list --filter="network:vpc3-custom"
gcloud compute firewall-rules describe vpc3-custom-allow-health-check

# 3. Verify Regional Health Check
gcloud compute health-checks list
gcloud compute health-checks describe regional-tcp-health-check --region=us-central1

# 4. Verify Instance Template
gcloud compute instance-templates list
gcloud compute instance-templates describe it-rlbdemo-us-central1

# 5. Verify Zonal MIGs
gcloud compute instance-groups managed list

echo "=== ZMIG 1 Details ==="
gcloud compute instance-groups managed describe zmig-us-1 --zone=us-central1-a

echo "=== ZMIG 2 Details ==="
gcloud compute instance-groups managed describe zmig-us-2 --zone=us-central1-c

# 6. Verify VM Instances
gcloud compute instances list --filter="name~'zmig-us-.*'"

# 7. Verify Health Status
echo "=== Health status for zmig-us-1 ==="
gcloud compute instance-groups managed list-instances zmig-us-1 \
    --zone=us-central1-a \
    --format="table(instance,status,healthState)"

echo "=== Health status for zmig-us-2 ==="
gcloud compute instance-groups managed list-instances zmig-us-2 \
    --zone=us-central1-c \
    --format="table(instance,status,healthState)"

# 8. Verify Named Ports
echo "=== Named ports configuration ==="
gcloud compute instance-groups describe zmig-us-1 --zone=us-central1-a --format="get(namedPorts)"
gcloud compute instance-groups describe zmig-us-2 --zone=us-central1-c --format="get(namedPorts)"
```

**Why are we doing this?**  
Comprehensive verification ensures everything is configured correctly:

**Expected results:**

1. ✅ **VPC Network**: vpc3-custom exists
2. ✅ **Subnet**: us-central1-subnet (10.135.0.0/20)
3. ✅ **Firewall Rules**: Health check rule allowing 130.211.0.0/22, 35.191.0.0/16
4. ✅ **Health Check**: regional-tcp-health-check on port 80
5. ✅ **Instance Template**: it-rlbdemo-us-central1 with nginix-vpc.sh
6. ✅ **ZMIGs**: 2 groups (zmig-us-1, zmig-us-2)
7. ✅ **VM Instances**: 4 total (2 per ZMIG)
8. ✅ **Health Status**: All instances HEALTHY
9. ✅ **Named Ports**: webserver80:80 configured

**If instances show UNHEALTHY:**
- Wait 5-10 minutes (initial delay + startup time)
- Check firewall rule for health check IPs
- Verify nginx is running on instances
- Check startup script executed successfully

---

## Step-09: Test the Deployment

**What we do:**
```bash
# Get external IPs of all ZMIG instances
echo "=== VM Instances and their External IPs ==="
gcloud compute instances list \
    --filter="name~'zmig-us-.*'" \
    --format="table(name,zone,networkInterfaces[0].networkIP,networkInterfaces[0].accessConfigs[0].natIP,status)"

# Test web access to each instance
echo "=== Testing Web Servers ==="
gcloud compute instances list \
    --filter="name~'zmig-us-.*'" \
    --format="value(networkInterfaces[0].accessConfigs[0].natIP)" | \
    while read ip; do 
        echo "Testing $ip"
        curl -s http://$ip | grep -E "(Hostname|IP Address|Kolejne demo)"
        echo "---"
    done

# SSH into an instance and test from within
INSTANCE_NAME=$(gcloud compute instances list --filter="name~'zmig-us-1-.*'" --format="value(name)" --limit=1)
gcloud compute ssh $INSTANCE_NAME --zone=us-central1-a

# Once connected, verify nginx is running:
sudo systemctl status nginx
curl localhost
exit
```

**Why are we doing this?**  
Testing confirms the deployment is working:

**Expected output:**
- ✅ 4 instances running (2 in each zone)
- ✅ Each instance has external IP
- ✅ Nginx responds on port 80
- ✅ Custom HTML shows "Kolejne demo za nami - Pozdrawiam Ania"
- ✅ Each instance shows unique hostname
- ✅ Internal IPs from 10.135.0.0/20 range

**What the test proves:**
1. Instances created successfully
2. Startup script executed (nginx installed)
3. Firewall rules allow HTTP traffic
4. Network configuration correct
5. Ready for load balancer attachment

---

## Step-10: Clean Up Resources

**What we do:**
```bash
# 1. Delete Zonal Managed Instance Groups (also deletes VM instances)
gcloud compute instance-groups managed delete zmig-us-1 \
    --zone=us-central1-a \
    --quiet

gcloud compute instance-groups managed delete zmig-us-2 \
    --zone=us-central1-c \
    --quiet

# 2. Delete Instance Template
gcloud compute instance-templates delete it-rlbdemo-us-central1 --quiet

# 3. Delete Regional Health Check
gcloud compute health-checks delete regional-tcp-health-check \
    --region=us-central1 \
    --quiet

# 4. (Optional) Delete firewall rule if not needed for other demos
# gcloud compute firewall-rules delete vpc3-custom-allow-health-check --quiet

# Verify cleanup
echo "=== Verifying Cleanup ==="
echo "Instance Groups:"
gcloud compute instance-groups managed list
echo "VM Instances:"
gcloud compute instances list
echo "Instance Templates:"
gcloud compute instance-templates list
echo "Health Checks:"
gcloud compute health-checks list
```

**Why are we doing this?**  
Proper cleanup prevents unnecessary charges:

**Cost implications:**
- **VM instances**: Charge per second while running (~$5/month per e2-micro)
- **External IPs**: ~$3/month per IP
- **Persistent disks**: Deleted with VMs automatically
- **Other resources**: No charge (templates, health checks, MIGs)

**Order of deletion:**
1. MIGs first (deletes VMs automatically)
2. Instance template (can't be deleted if MIGs use it)
3. Health check (can be deleted anytime)
4. Firewall rules (keep if needed for other demos)

---

## Summary

In this tutorial, you learned:
- **Zonal MIGs**: Create managed instance groups in specific zones
- **Regional Health Checks**: TCP-based health monitoring for regional resources
- **Manual Multi-Zone**: Create separate MIGs in each zone for HA
- **Auto-Healing**: Automatically replace unhealthy instances
- **Named Ports**: Configure for load balancer integration

**Key Concepts:**

| Component | Purpose | Benefit |
|-----------|---------|---------|
| **Zonal MIG** | Manage instances in a zone | Fine-grained control |
| **Regional Health Check** | Monitor instance health | Enable auto-healing |
| **Instance Template** | VM blueprint | Consistent configuration |
| **Named Ports** | Label service ports | Load balancer integration |
| **Firewall Rules** | Allow health checks | Critical for MIG operation |

**Architecture Achieved:**

```
Region: us-central1
├─ Zone: us-central1-a
│  └─ zmig-us-1 (Zonal MIG)
│     ├─ Instance 1 (nginx)
│     └─ Instance 2 (nginx)
└─ Zone: us-central1-c
   └─ zmig-us-2 (Zonal MIG)
      ├─ Instance 1 (nginx)
      └─ Instance 2 (nginx)

Total: 2 ZMIGs, 4 instances, 2 zones
```

**Zonal vs Regional MIG Decision Guide:**

**Use Zonal MIGs when:**
- ✅ You need Regional Load Balancer (not Global)
- ✅ You want different instance counts per zone
- ✅ You need fine-grained control over each zone
- ✅ You're optimizing costs per zone
- ✅ You have zone-specific requirements

**Use Regional MIGs when:**
- ✅ You need Global Load Balancer
- ✅ You want automatic zone distribution
- ✅ You prefer simplified management
- ✅ High availability is the priority
- ✅ You don't need per-zone customization

**Next Steps:**

After mastering Zonal MIGs, you can:
- Create a Regional Load Balancer using these ZMIGs
- Implement auto-scaling per zone
- Configure different health checks per ZMIG
- Set up monitoring and alerting
- Deploy in additional zones for more redundancy

**Production Best Practices:**

1. **Always use health checks**: Critical for auto-healing
2. **Set appropriate initial delay**: Allow time for startup
3. **Use at least 2 zones**: For high availability
4. **Configure named ports**: Required for load balancing
5. **Monitor MIG metrics**: Track health, scaling, errors
6. **Use network tags**: For firewall and organization
7. **Version templates**: Track configuration changes

**Common Use Cases for Zonal MIGs:**

1. **Regional applications**: Serve users in a specific region
2. **Data residency**: Keep data in specific zones
3. **Cost optimization**: Control costs per zone
4. **Legacy migration**: Move from single-zone to multi-zone gradually
5. **Zone-specific configs**: Different settings per zone

**Key Takeaway:** Zonal MIGs provide fine-grained control over instance placement and management within specific zones. They're ideal for Regional Load Balancers and scenarios requiring per-zone customization, while Regional MIGs are better for simplified management with automatic zone distribution for Global Load Balancers.
