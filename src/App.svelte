<script>
  import Book from "./lib/Book.svelte"
  import { onMount } from 'svelte';

  let showBook = false;
    let caption = "The World became a little brighter today, 13 years ago.";
  onMount(() => {
           // Create sparkles
        function createSparkles() {
            for (let i = 0; i < 30; i++) {
                const sparkle = document.createElement('div');
                sparkle.className = 'sparkle';
                sparkle.style.left = Math.random() * 100 + '%';
                sparkle.style.top = Math.random() * 100 + '%';
                sparkle.style.animationDelay = Math.random() * 2 + 's';
                sparkle.dataset.baseLeft = sparkle.style.left;
                document.querySelector('.container').appendChild(sparkle);
            }
        }
        createSparkles();
        
        // Update sparkles position based on camera
        function updateSparkles() {
            const sparkles = document.querySelectorAll('.sparkle');
            sparkles.forEach((sparkle, i) => {
                const baseLeft = parseFloat(sparkle.dataset.baseLeft);
                const newLeft = baseLeft - (camera.x / canvas.width) * 10;
                sparkle.style.left = newLeft + '%';
            });
            
            // Toggle body class for background animation
            if (stickman.state === 'running') {
                document.body.classList.add('running');
            } else {
                document.body.classList.remove('running');
            }
        }
        
        const canvas = document.getElementById('treeCanvas');
        const ctx = canvas.getContext('2d');
        
        canvas.width = 800;
        canvas.height = 600;
        
        const groundY = canvas.height - 50;
        
        let seed = {
            x: canvas.width / 2,
            y: 50,
            radius: 5,
            velocityY: 0,
            gravity: 0.3,
            landed: false,
            bounces: 0,
            maxBounces: 3,
            bounceDamping: 0.6
        };
        
        let tree = {
            x: canvas.width / 2,
            y: groundY,
            height: 0,
            maxHeight: 60,
            growing: false,
            branches: [],
            leaves: [],
            trunkCurveX: (Math.random() - 0.5) * 10,
            trunkCurveY: 10
        };
        
        let animationPhase = 'falling';
        
        let stickman = {
            visible: false,
            headRadius: 12,
            bodyLength: 35,
            armLength: 25,
            legLength: 30,
            color: '#87CEEB', // Sky blue
            state: 'sitting', // sitting, standing, running, throwing, frozen
            standingProgress: 0,
            runCycle: 0, // For animating running
            startRunningTime: null,
            throwingProgress: 0
        };
        
        let book = {
            visible: false,
            x: 0,
            y: 0,
            velocityX: 15,
            velocityY: -8,
            gravity: 0.4,
            rotation: 0,
            rotationSpeed: 0.3,
            width: 20,
            height: 15
        };
        
        let animationFrozen = false;
        
        let camera = {
            x: 0, // Camera offset for scrolling effect
            targetX: 0,
            speed: 7 // Decreased from 6 to 4
        };
        
        let rocks = [];
        let throwingRocks = false;
        
        class Rock {
            constructor() {
                this.x = -20; // Start from left side, off screen
                this.y = Math.random() * 50 + 400; // Lower spawn position (400-450)
                this.velocityX = 8 + Math.random() * 4; // Much faster horizontal speed
                this.velocityY = -3 - Math.random() * 2; // Initial upward velocity
                this.gravity = 0.15; // Gravity pulling down
                this.radius = 4 + Math.random() * 4; // Slightly larger
                this.rotation = 0;
                this.rotationSpeed = (Math.random() - 0.5) * 0.2;
                this.color = '#696969'; // Dark gray
                this.shape = this.generateRockShape(); // Pre-generate shape
            }
            
            generateRockShape() {
                const points = 7 + Math.floor(Math.random() * 3);
                const angles = [];
                for (let i = 0; i < points; i++) {
                    angles.push({
                        angle: (i / points) * Math.PI * 2,
                        radius: this.radius * (0.6 + Math.random() * 0.4)
                    });
                }
                return angles;
            }
            
            update() {
                this.x += this.velocityX;
                this.velocityY += this.gravity; // Apply gravity
                this.y += this.velocityY;
                this.rotation += this.rotationSpeed;
                
                // Remove if off screen
                return this.x < canvas.width + 50 && this.y < canvas.height + 50;
            }
            
            draw() {
                ctx.save();
                ctx.translate(this.x, this.y);
                ctx.rotate(this.rotation);
                
                // Draw irregular rock shape with consistent edges
                ctx.fillStyle = this.color;
                ctx.strokeStyle = '#4a4a4a';
                ctx.lineWidth = 1;
                
                ctx.beginPath();
                this.shape.forEach((point, i) => {
                    const x = Math.cos(point.angle) * point.radius;
                    const y = Math.sin(point.angle) * point.radius;
                    if (i === 0) {
                        ctx.moveTo(x, y);
                    } else {
                        ctx.lineTo(x, y);
                    }
                });
                ctx.closePath();
                ctx.fill();
                ctx.stroke();
                
                ctx.restore();
            }
        }
        
        class Branch {
            constructor(startX, startY, angle, length, depth) {
                this.startX = startX;
                this.startY = startY;
                this.angle = angle;
                this.length = length;
                this.depth = depth;
                this.currentLength = 0;
                this.growing = false;
                
                const baseEndX = startX + Math.cos(angle) * length;
                const baseEndY = startY + Math.sin(angle) * length;
                
                this.curveOffsetX = (Math.random() - 0.5) * 20;
                this.curveOffsetY = (Math.random() - 0.5) * 20;
                
                this.endX = baseEndX;
                this.endY = baseEndY;
            }
            
            grow() {
                if (this.currentLength < this.length) {
                    this.currentLength += 2;
                    return false;
                }
                return true;
            }
            
            draw() {
                const progress = this.currentLength / this.length;
                const x = this.startX + Math.cos(this.angle) * this.currentLength;
                const y = this.startY + Math.sin(this.angle) * this.currentLength;
                
                ctx.strokeStyle = '#ffabfc';
                ctx.lineWidth = Math.max(1, 8 - this.depth * 1.5);
                ctx.beginPath();
                ctx.moveTo(this.startX, this.startY);
                
                const midX = (this.startX + x) / 2 + this.curveOffsetX * progress;
                const midY = (this.startY + y) / 2 + this.curveOffsetY * progress;
                ctx.quadraticCurveTo(midX, midY, x, y);
                
                ctx.stroke();
            }
        }
        
        class Leaf {
            constructor(x, y) {
                this.x = x;
                this.y = y;
                this.size = 0;
                this.maxSize = 8 + Math.random() * 4;
                this.color = this.getRandomColor();
                this.growing = true;
            }
            
            getRandomColor() {
                const colors = ['#FF69B4', '#FF1493', '#FF6347', '#FF4500', '#FFA500', '#FFD700', '#9370DB', '#BA55D3'];
                return colors[Math.floor(Math.random() * colors.length)];
            }
            
            grow() {
                if (this.size < this.maxSize) {
                    this.size += 0.3;
                    return false;
                }
                return true;
            }
            
            draw() {
                ctx.fillStyle = this.color;
                ctx.beginPath();
                
                const x = this.x;
                const y = this.y;
                const size = this.size;
                
                ctx.moveTo(x, y + size / 4);
                ctx.bezierCurveTo(x, y, x - size / 2, y - size / 2, x - size, y + size / 4);
                ctx.bezierCurveTo(x - size, y + size, x, y + size * 1.5, x, y + size * 1.8);
                ctx.bezierCurveTo(x, y + size * 1.5, x + size, y + size, x + size, y + size / 4);
                ctx.bezierCurveTo(x + size / 2, y - size / 2, x, y, x, y + size / 4);
                
                ctx.fill();
            }
        }

        function createBranches(x, y, angle, length, depth, maxDepth) {
            if (depth > maxDepth) return;
            
            const branch = new Branch(x, y, angle, length, depth);
            tree.branches.push(branch);
            
            if (depth < maxDepth) {
                const endX = x + Math.cos(angle) * length;
                const endY = y + Math.sin(angle) * length;
                
                const numBranches = depth === 0 ? 4 : (3 + Math.floor(Math.random() * 2));
                for (let i = 0; i < numBranches; i++) {
                    const angleOffset = (Math.random() - 0.5) * Math.PI / 1.3;
                    const newAngle = angle + angleOffset;
                    const newLength = length * (0.65 + Math.random() * 0.15);
                    createBranches(endX, endY, newAngle, newLength, depth + 1, maxDepth);
                }
            }
        }
        
        function drawGround() {
            ctx.strokeStyle = '#ffffff';
            ctx.lineWidth = 3;
            ctx.beginPath();
            // Draw ground line extending way beyond visible area
            ctx.moveTo(-10000, groundY);
            ctx.lineTo(50000, groundY);
            ctx.stroke();
        }
        
        function drawStickman() {
            if (!stickman.visible) return;
            
            // Stickman is always at center of screen
            const baseX = canvas.width / 2;
            const baseY = groundY;
            
            ctx.strokeStyle = stickman.color;
            ctx.fillStyle = stickman.color;
            ctx.lineWidth = 3;
            ctx.lineCap = 'round';
            ctx.lineJoin = 'round';
            
            if (stickman.state === 'sitting') {
                // Original sitting pose
                const pelvisY = baseY - 3;
                const pelvisX = baseX;
                
                // Head
                const headX = pelvisX;
                const headY = pelvisY - stickman.bodyLength - stickman.headRadius;
                ctx.beginPath();
                ctx.arc(headX, headY, stickman.headRadius, 0, Math.PI * 2);
                ctx.stroke();
                
                // Body
                ctx.beginPath();
                ctx.moveTo(headX, headY + stickman.headRadius);
                ctx.lineTo(pelvisX, pelvisY);
                ctx.stroke();
                
                // Legs (folded)
                const leftKneeX = pelvisX - 25;
                const leftKneeY = pelvisY - 25;
                const leftAnkleX = pelvisX - 35;
                const leftAnkleY = baseY;
                
                ctx.beginPath();
                ctx.moveTo(pelvisX, pelvisY);
                ctx.lineTo(leftKneeX, leftKneeY);
                ctx.stroke();
                
                ctx.beginPath();
                ctx.moveTo(leftKneeX, leftKneeY);
                ctx.lineTo(leftAnkleX, leftAnkleY);
                ctx.stroke();
                
                const rightKneeX = pelvisX - 30;
                const rightKneeY = pelvisY - 22;
                const rightAnkleX = pelvisX - 42;
                const rightAnkleY = baseY;
                
                ctx.beginPath();
                ctx.moveTo(pelvisX, pelvisY);
                ctx.lineTo(rightKneeX, rightKneeY);
                ctx.stroke();
                
                ctx.beginPath();
                ctx.moveTo(rightKneeX, rightKneeY);
                ctx.lineTo(rightAnkleX, rightAnkleY);
                ctx.stroke();
                
                // Arms (elbows on knees)
                const leftShoulderX = headX - 3;
                const leftShoulderY = headY + stickman.headRadius + 5;
                const leftElbowX = leftKneeX;
                const leftElbowY = leftKneeY;
                const leftHandX = leftElbowX - 8;
                const leftHandY = leftElbowY - 15;
                
                ctx.beginPath();
                ctx.moveTo(leftShoulderX, leftShoulderY);
                ctx.lineTo(leftElbowX, leftElbowY);
                ctx.stroke();
                
                ctx.beginPath();
                ctx.moveTo(leftElbowX, leftElbowY);
                ctx.lineTo(leftHandX, leftHandY);
                ctx.stroke();
                
                const rightShoulderX = headX + 3;
                const rightShoulderY = headY + stickman.headRadius + 5;
                const rightElbowX = rightKneeX;
                const rightElbowY = rightKneeY;
                const rightHandX = rightElbowX - 5;
                const rightHandY = rightElbowY - 15;
                
                ctx.beginPath();
                ctx.moveTo(rightShoulderX, rightShoulderY);
                ctx.lineTo(rightElbowX, rightElbowY);
                ctx.stroke();
                
                ctx.beginPath();
                ctx.moveTo(rightElbowX, rightElbowY);
                ctx.lineTo(rightHandX, rightHandY);
                ctx.stroke();
                
                // Book
                const bookCenterX = (leftHandX + rightHandX) / 2;
                const bookCenterY = (leftHandY + rightHandY) / 2 + 8;
                const bookWidth = 20;
                const bookHeight = 15;
                
                ctx.fillStyle = '#FFE4B5';
                ctx.fillRect(bookCenterX - bookWidth/2, bookCenterY - bookHeight/2, bookWidth, bookHeight);
                
                ctx.strokeStyle = '#8B4513';
                ctx.lineWidth = 2;
                ctx.strokeRect(bookCenterX - bookWidth/2, bookCenterY - bookHeight/2, bookWidth, bookHeight);
                
                ctx.beginPath();
                ctx.moveTo(bookCenterX, bookCenterY - bookHeight/2);
                ctx.lineTo(bookCenterX, bookCenterY + bookHeight/2);
                ctx.stroke();
                
            } else if (stickman.state === 'standing') {
                // Standing up animation
                stickman.standingProgress += 0.06;
                if (stickman.standingProgress >= 1) {
                    stickman.state = 'running';
                    stickman.startRunningTime = Date.now();
                }
                
                const progress = Math.min(stickman.standingProgress, 1);
                const easeProgress = 1 - Math.pow(1 - progress, 3); // Ease out cubic
                
                // Interpolate from sitting to standing
                const pelvisY = baseY - 3 - easeProgress * 30;
                const pelvisX = baseX;
                
                const headX = pelvisX;
                const headY = pelvisY - stickman.bodyLength - stickman.headRadius;
                
                ctx.beginPath();
                ctx.arc(headX, headY, stickman.headRadius, 0, Math.PI * 2);
                ctx.stroke();
                
                ctx.beginPath();
                ctx.moveTo(headX, headY + stickman.headRadius);
                ctx.lineTo(pelvisX, pelvisY);
                ctx.stroke();
                
                // Legs straightening
                const legStraightness = easeProgress;
                
                ctx.beginPath();
                ctx.moveTo(pelvisX, pelvisY);
                ctx.lineTo(pelvisX - 5, baseY);
                ctx.stroke();
                
                ctx.beginPath();
                ctx.moveTo(pelvisX, pelvisY);
                ctx.lineTo(pelvisX + 5, baseY);
                ctx.stroke();
                
                // Arms going to sides
                const shoulderY = headY + stickman.headRadius + 5;
                
                ctx.beginPath();
                ctx.moveTo(headX, shoulderY);
                ctx.lineTo(headX - 10 - easeProgress * 10, shoulderY + 15);
                ctx.stroke();
                
                ctx.beginPath();
                ctx.moveTo(headX, shoulderY);
                ctx.lineTo(headX + 10 + easeProgress * 10, shoulderY + 15);
                ctx.stroke();
                
            } else if (stickman.state === 'running') {
                // Check if it's time to throw the book
                if (!stickman.startRunningTime) {
                    stickman.startRunningTime = Date.now();
                }
                const runningDuration = (Date.now() - stickman.startRunningTime) / 1000;
                if (runningDuration >= 5 && !book.visible) {
                    stickman.state = 'throwing';
                    stickman.throwingProgress = 0;
                }
                
                // Running animation - proper running form with knees facing right
                stickman.runCycle += 0.9; // Changed from += 1 to be frame-rate independent
                
                const bobAmount = Math.abs(Math.sin(stickman.runCycle)) * 2;
                const pelvisY = baseY - 33 + bobAmount;
                const pelvisX = baseX;
                
                const headX = pelvisX+10;
                const headY = pelvisY - stickman.bodyLength - stickman.headRadius;
                
                // Head (bobbing)
                ctx.beginPath();
                ctx.arc(headX, headY, stickman.headRadius, 0, Math.PI * 2);
                ctx.stroke();
                
                // Body (leaning forward while running)
                ctx.beginPath();
                ctx.moveTo(headX, headY + stickman.headRadius);
                ctx.lineTo(pelvisX + 5, pelvisY);
                ctx.stroke();
                
                // Running legs - knees moving forward to the right
                const legPhase1 = Math.sin(stickman.runCycle);
                const legPhase2 = Math.sin(stickman.runCycle + Math.PI);
                
                // Left leg - knee extends to the right
                const leftKneeForward = 10 + legPhase1 * 15;
                const leftKneeX = pelvisX + leftKneeForward;
                const leftKneeY = pelvisY + 13 - Math.abs(legPhase1) * 13;
                const leftFootX = leftKneeX + 15 - Math.abs(legPhase1) * 8;
                const leftFootY = baseY;
                
                ctx.beginPath();
                ctx.moveTo(pelvisX+5, pelvisY);
                ctx.lineTo(leftKneeX, leftFootY);
                ctx.stroke();
                
                // Right leg - knee extends to the right
                const rightKneeForward = 10 + legPhase2 * 15;
                const rightKneeX = pelvisX + rightKneeForward;
                const rightKneeY = pelvisY + 13 - Math.abs(legPhase2) * 13;
                const rightFootX = rightKneeX - 15 - Math.abs(legPhase2) * 8;
                const rightFootY = baseY;
                
                ctx.beginPath();
                ctx.moveTo(pelvisX+5, pelvisY);
                ctx.lineTo(rightKneeX, rightFootY);
                ctx.stroke();
                
                // Running arms - swinging forward (opposite to legs)
                const armPhase1 = -legPhase1 * 0.5;
                const armPhase2 = -legPhase2 * 0.5;
                
                const shoulderY = headY + stickman.headRadius + 5;
                
                // Left arm
                const leftElbowX = headX + 10 + Math.sin(armPhase1) * 15;
                const leftElbowY = shoulderY + 10 + Math.abs(Math.cos(armPhase1)) * 5;
                const leftHandX = leftElbowX + 12;
                const leftHandY = leftElbowY + 8;
                
                ctx.beginPath();
                ctx.moveTo(headX - 3, shoulderY);
                ctx.lineTo(leftElbowX, leftElbowY);
                ctx.lineTo(leftHandX, leftHandY);
                ctx.stroke();
                
                // Right arm
                const rightElbowX = headX + 10 + Math.sin(armPhase2) * 15;
                const rightElbowY = shoulderY + 10 + Math.abs(Math.cos(armPhase2)) * 5;
                const rightHandX = rightElbowX + 12;
                const rightHandY = rightElbowY + 8;
                
                ctx.beginPath();
                ctx.moveTo(headX + 3, shoulderY);
                ctx.lineTo(rightElbowX, rightElbowY);
                ctx.lineTo(rightHandX, rightHandY);
                ctx.stroke();
                // Move camera to follow
                camera.targetX += camera.speed;
                
            } else if (stickman.state === 'throwing') {
                // Throwing animation
                stickman.throwingProgress += 0.05;
                
                const pelvisY = baseY - 33;
                const pelvisX = baseX;
                
                const headX = pelvisX;
                const headY = pelvisY - stickman.bodyLength - stickman.headRadius;
                
                // Head
                ctx.beginPath();
                ctx.arc(headX, headY, stickman.headRadius, 0, Math.PI * 2);
                ctx.stroke();
                
                // Body
                ctx.beginPath();
                ctx.moveTo(headX, headY + stickman.headRadius);
                ctx.lineTo(pelvisX, pelvisY);
                ctx.stroke();
                
                // Legs standing
                ctx.beginPath();
                ctx.moveTo(pelvisX, pelvisY);
                ctx.lineTo(pelvisX - 5, baseY);
                ctx.stroke();
                
                ctx.beginPath();
                ctx.moveTo(pelvisX, pelvisY);
                ctx.lineTo(pelvisX + 5, baseY);
                ctx.stroke();
                
                const shoulderY = headY + stickman.headRadius + 5;
                
                // Throwing arm motion
                const throwAngle = Math.min(stickman.throwingProgress * Math.PI, Math.PI * 0.4);
                
                // Right arm (throwing)
                const rightElbowX = headX + 15 + Math.cos(throwAngle) * 20;
                const rightElbowY = shoulderY - 10 - Math.sin(throwAngle) * 15;
                const rightHandX = rightElbowX + Math.cos(throwAngle) * 15;
                const rightHandY = rightElbowY - Math.sin(throwAngle) * 10;
                
                ctx.beginPath();
                ctx.moveTo(headX + 3, shoulderY);
                ctx.lineTo(rightElbowX, rightElbowY);
                ctx.lineTo(rightHandX, rightHandY);
                ctx.stroke();
                
                // Left arm (steady)
                ctx.beginPath();
                ctx.moveTo(headX - 3, shoulderY);
                ctx.lineTo(headX - 10, shoulderY + 15);
                ctx.lineTo(headX - 8, shoulderY + 25);
                ctx.stroke();
                
                // Launch book when throw is complete
                if (stickman.throwingProgress >= 0.3 && !book.visible) {
                    book.visible = true;
                    book.x = baseX + 20;
                    book.y = headY + 10;
                    book.rotation = 0;
                    stickman.state = 'frozen';
                }
                
                // Stop camera
                // Don't move camera during throw
            }
        }
        
        function drawSeed() {
            ctx.fillStyle = '#ffabfc';
            ctx.beginPath();
            ctx.arc(seed.x, seed.y, seed.radius, 0, Math.PI * 2);
            ctx.fill();
        }
        
        function updateSeed() {
            if (!seed.landed) {
                seed.velocityY += seed.gravity;
                seed.y += seed.velocityY;
                
                if (seed.y + seed.radius >= groundY) {
                    seed.y = groundY - seed.radius;
                    
                    if (seed.bounces < seed.maxBounces) {
                        seed.velocityY = -seed.velocityY * seed.bounceDamping;
                        seed.bounces++;
                    } else {
                        seed.landed = true;
                        seed.velocityY = 0;
                        setTimeout(() => {
                            animationPhase = 'growing';
                            tree.growing = true;
                        }, 500);
                    }
                }
            }
        }
        
        function drawTrunk() {
            ctx.strokeStyle = '#ffabfc';
            ctx.lineWidth = 8;
            ctx.beginPath();
            ctx.moveTo(tree.x, tree.y);
            
            const endX = tree.x;
            const endY = tree.y - tree.height;
            const controlX = tree.x + tree.trunkCurveX;
            const controlY = tree.y - tree.height / 2;
            
            ctx.quadraticCurveTo(controlX, controlY, endX, endY);
            ctx.stroke();
        }
        
        function updateTree() {
            if (tree.growing && tree.height < tree.maxHeight) {
                tree.height += 2;
                
                if (tree.height >= tree.maxHeight) {
                    tree.growing = false;
                    animationPhase = 'branching';
                    createBranches(tree.x, tree.y - tree.height, -Math.PI / 2, 120, 0, 6);
                    tree.branches.sort((a, b) => b.startY - a.startY);
                    tree.branches[0].growing = true;
                }
            }
        }
        
        function updateBranches() {
            let allBranchesGrown = true;
            
            for (let i = 0; i < tree.branches.length; i++) {
                const branch = tree.branches[i];
                
                if (branch.growing) {
                    const finished = branch.grow();
                    if (finished) {
                        branch.growing = false;
                        const threshold = 2;
                        for (let j = 0; j < tree.branches.length; j++) {
                            if (i !== j && !tree.branches[j].growing && tree.branches[j].currentLength === 0) {
                                const distToStart = Math.sqrt(
                                    Math.pow(tree.branches[j].startX - branch.endX, 2) +
                                    Math.pow(tree.branches[j].startY - branch.endY, 2)
                                );
                                if (distToStart < threshold) {
                                    tree.branches[j].growing = true;
                                }
                            }
                        }
                    }
                }
                
                if (branch.currentLength < branch.length) {
                    allBranchesGrown = false;
                }
            }
            
            if (allBranchesGrown && animationPhase === 'branching') {
                animationPhase = 'leaves';
                tree.branches.forEach(branch => {
                    if (branch.depth >= 1) {
                        const endX = branch.startX + Math.cos(branch.angle) * branch.length;
                        const endY = branch.startY + Math.sin(branch.angle) * branch.length;
                        const numLeaves = 4 + Math.floor(Math.random() * 3);
                        for (let i = 0; i < numLeaves; i++) {
                            const leafAngle = (i / numLeaves) * Math.PI * 2;
                            const radius = 10 + Math.random() * 15;
                            const offsetX = Math.cos(leafAngle) * radius;
                            const offsetY = Math.sin(leafAngle) * radius;
                            tree.leaves.push(new Leaf(endX + offsetX, endY + offsetY));
                        }
                    }
                });
                // Show stickman and immediately start throwing rocks
                setTimeout(() => {
                    stickman.visible = true;
                    throwingRocks = true;
                }, 500); // Reduced from 800ms to 500ms
            }
        }
        
        function updateLeaves() {
            tree.leaves.forEach(leaf => {
                if (leaf.growing) {
                    leaf.grow();
                }
            });
        }
        
        function updateRocks() {
            if (throwingRocks && !animationFrozen) {
                // Add new rock more frequently (higher spawn rate)
                if (Math.random() < 0.08) {
                    rocks.push(new Rock());
                }
                
                // Check if stickman should get scared (when a rock gets close)
                if (stickman.state === 'sitting' && rocks.length > 3) {
                    const stickmanX = canvas.width / 2 - camera.x;
                    for (let rock of rocks) {
                        if (rock.x > stickmanX - 100 && rock.x < stickmanX + 50) {
                            stickman.state = 'standing';
                            break;
                        }
                    }
                }
            }
            
            // Update existing rocks
            if (!animationFrozen) {
                rocks = rocks.filter(rock => rock.update());
            }
        }
        
        function updateBook() {
            if (book.visible && !animationFrozen) {
                book.velocityY += book.gravity;
                book.x += book.velocityX;
                book.y += book.velocityY;
                book.rotation += book.rotationSpeed;
                
                // Check collision with right side of screen
                if (book.x >= canvas.width - 20) {
                    animationFrozen = true;
                    freezed();
                }
            }
        }
        
        function drawBook() {
            if (book.visible) {
                ctx.save();
                ctx.translate(book.x, book.y);
                ctx.rotate(book.rotation);
                
                // Draw book
                ctx.fillStyle = '#FFE4B5';
                ctx.fillRect(-book.width/2, -book.height/2, book.width, book.height);
                
                ctx.strokeStyle = '#8B4513';
                ctx.lineWidth = 2;
                ctx.strokeRect(-book.width/2, -book.height/2, book.width, book.height);
                
                ctx.beginPath();
                ctx.moveTo(0, -book.height/2);
                ctx.lineTo(0, book.height/2);
                ctx.stroke();
                
                ctx.restore();
            }
        }
        
        function freezed() {
            let opa = document.getElementById("openbook").style.opacity = 100;
            // alert("WORK IN PROGRESS> MAIN STUFF STARTS HERE");
        }
        
        function drawRocks() {
            rocks.forEach(rock => rock.draw());
        }
        
        function draw() {
            ctx.clearRect(0, 0, canvas.width, canvas.height);
            
            // Smooth camera movement
            if (!animationFrozen) {
                camera.x += (camera.targetX - camera.x) * 0.1;
            }
            
            // Save context and apply camera transform
            ctx.save();
            ctx.translate(-camera.x, 0);
            
            drawGround();
            
            if (animationPhase === 'falling') {
                drawSeed();
            }
            
            if (animationPhase === 'growing' || animationPhase === 'branching' || animationPhase === 'leaves') {
                drawTrunk();
            }
            
            if (animationPhase === 'branching' || animationPhase === 'leaves') {
                tree.branches.forEach(branch => branch.draw());
            }
            
            if (animationPhase === 'leaves') {
                tree.leaves.forEach(leaf => leaf.draw());
            }
            
            // Draw rocks
            drawRocks();
            
            ctx.restore();
            
            // Draw stickman without camera offset (stays at center)
            drawStickman();
            
            // Draw book without camera offset (moves independently)
            drawBook();
        }
        
        // Frame-rate independent animation using delta time
        let lastTime = performance.now();
        const targetFPS = 60;
        const frameTime = 1000 / targetFPS;
        function updateCaption(){
            caption = animationPhase === 'growing'||animationPhase === 'branching' ? "Look at that beutiful tree growing..." :caption;
            if(animationPhase === 'leaves'){
                caption = stickman.state === 'sitting'|| stickman.state === 'standing'? "They started throwing rocks? Oh god look at that stickman, he must be scared" :
                      stickman.state === 'running' ? "Oh he is running away from the rocks! Run sticky run!" :
                      stickman.state === 'throwing'||stickman.state === "frozen" ? "What is in that book?":'';
            }
                
        }
        function animate(currentTime) {
            const deltaTime = currentTime - lastTime;
            
            // Only update if enough time has passed (frame limiting to 60fps)
            if (deltaTime >= frameTime) {
                lastTime = currentTime - (deltaTime % frameTime);
                
                if (!animationFrozen) {
                    if (animationPhase === 'falling') {
                        updateSeed();
                    } else if (animationPhase === 'growing') {
                        updateTree();
                    } else if (animationPhase === 'branching') {
                        updateBranches();
                    } else if (animationPhase === 'leaves') {
                        updateLeaves();
                    }
                    
                    // Always update rocks once throwing starts
                    updateRocks();
                    
                    // Update book physics
                    updateBook();
                    
                    // Update sparkles position
                    updateSparkles();
                    
                    //Update Captions
                    updateCaption();
                }   
                
                
                draw();
            }
            
            requestAnimationFrame(animate);
        }
        animate(performance.now());
        // freezed()
  });

  function handleOpenBook() {
    showBook = true;
    document.getElementById("openbook").style.opacity = 0;

  }
</script>

<svelte:head>
  <link rel="preconnect" href="https://fonts.googleapis.com">
  <link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
  <link href="https://fonts.googleapis.com/css2?family=Poppins:ital,wght@0,100;0,200;0,300;0,400;0,500;0,600;0,700;0,800;0,900;1,100;1,200;1,300;1,400;1,500;1,600;1,700;1,800;1,900&display=swap" rel="stylesheet">
</svelte:head>

<div class="container">
    <h1 class="hbd">Just a simple animation for you 💗</h1>
    <canvas id="treeCanvas"></canvas>
    <div class="text-container">
        <div class="text-main">{caption}</div>
    </div>
    
    <div class="openbook" id="openbook" style="opacity:0;">
        The stickman threw a book for you to read.
        <br/>Shall I open the book?  
        <button on:click={handleOpenBook}>Sure</button>  
    </div>

    {#if showBook}
        <div class="book">
            <div class="book-item">
            <Book/>

            </div>
        </div>
    {/if}
</div>

<style>
    :global(body) {
        margin: 0;
        padding: 0;
        box-sizing: border-box;
        font-family: 'Poppins', sans-serif;
    }
    .book{
        height: 100vh;
        width:100vw;
        display:flex;
        align-items: center;
        justify-content: center;
        position:absolute;
        z-index:10;
    }
    .book-item{
        position: relative;
        z-index:20;
        width:60%;
    }
    .container {
        position: relative;
        width: 100vw;
        height: 100vh;
        display: flex;
        justify-content: center;
        align-items: center;
        background: linear-gradient(0deg, #1f011a 20%, #02011f 80%);
        overflow: hidden;
    }
    
    #treeCanvas {
        position: absolute;
        top: 50%;
        left: 50%;
        transform: translate(-50%, -50%);
        z-index: 1;
        width:100vw;
        max-width: 800px;
    }
    
    .hbd {
        position: absolute;
        top: 40px;
        left: 50%;
        transform: translateX(-50%);
        color: white;
        font-weight: 600;
        font-size: 20px;
        text-shadow: 
            0px 0px 10px rgba(0, 0, 0, 0.8),
            0px 0px 20px rgba(0, 0, 0, 0.6),
            0px 0px 30px rgba(0, 0, 0, 0.4);
        z-index: 10;
        animation: fadeInDown 1s ease-out;
    }
    
    .text-container {
        position: absolute;
        bottom: 20%;
        left: 50%;
        transform: translateX(-50%);
        text-align: center;
        z-index: 10;
        animation: fadeInUp 1.5s ease-out;
        width:100vw;
    }
    
    .text-main {
        color: white;
        font-weight: 400;
        font-size: 1.3rem;
        text-shadow: 
            0px 2px 4px rgba(0, 0, 0, 0.5),
            0px 4px 8px rgba(0, 0, 0, 0.3);
        line-height: 1.6;
    }
    
    @keyframes fadeInDown {
        from {
            opacity: 0;
            transform: translateX(-50%) translateY(-30px);
        }
        to {
            opacity: 1;
            transform: translateX(-50%) translateY(0);
        }
    }
    
    @keyframes fadeInUp {
        from {
            opacity: 0;
            transform: translateX(-50%) translateY(30px);
        }
        to {
            opacity: 1;
            transform: translateX(-50%) translateY(0);
        }
    }
    
    .openbook{
        background-color: hsl(0, 100%, 90%);
        height:auto;
        padding:30px 0px;
        width: 80vw;
        border: 6px solid pink;
        border-radius: 20px;
        display: flex;
        align-items: center;
        justify-content: center;
        font-family: 'Poppins', sans-serif;
        font-size: clamp(15px, 2vw, 30px);
        text-align: center;
        flex-direction: column;
        z-index: 5;
        transition: opacity 0.3s;
    }
    
    .openbook button{
        padding:10px 20px;
        margin-top: 20px;
        font-size: 1.2rem;
        border-radius: 9999px;
        border: 6px solid pink;
        cursor: pointer;
        transition: all 0.3s;
    }

    .openbook button:hover{
        padding:20px 30px;
    }
    
    @media (max-width: 768px) {
        .hbd {
            font-size: 1rem;
            top: 30px;
        }
        .text-main {
            font-size: 1rem;
        }
    }
</style>