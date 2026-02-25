<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Growing Tree Animation</title>
    <style>
        body {
            margin: 0;
            padding: 0;
            display: flex;
            justify-content: center;
            align-items: center;
            min-height: 100vh;
            background: url("/img2.png");
             background-position: center;
            
            background-repeat: no-repeat;
        }
        #treeCanvas{
            width:100vw;
        }
        
    </style>
</head>

<body>
    <canvas id="treeCanvas"></canvas>
    <span class="Text">The World Became a Better Place 13 years ago, Today</span>
    <script>
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
        
        let animationPhase = 'falling'; // falling, growing, branching, leaves
        
        class Branch {
            constructor(startX, startY, angle, length, depth) {
                this.startX = startX;
                this.startY = startY;
                this.angle = angle;
                this.length = length;
                this.depth = depth;
                this.currentLength = 0;
                this.growing = false;
                
                // Calculate base end position
                const baseEndX = startX + Math.cos(angle) * length;
                const baseEndY = startY + Math.sin(angle) * length;
                
                // Store curve control points for consistent curved branches
                this.curveOffsetX = (Math.random() - 0.5) * 20;
                this.curveOffsetY = (Math.random() - 0.5) * 20;
                
                // The actual end position is just the straight line end
                // Child branches will use this as their start
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
                
                // Create curved branches using stored control points
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
                
                // Draw heart shape
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
                
                // Always create 3 or 4 sub-branches
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
            ctx.moveTo(0, groundY);
            ctx.lineTo(canvas.width, groundY);
            ctx.stroke();
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
                        // Bounce back up
                        seed.velocityY = -seed.velocityY * seed.bounceDamping;
                        seed.bounces++;
                    } else {
                        // Stop bouncing and start growing
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
            
            // Draw curved trunk
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
                    // Sort branches by startY (bottom to top) so lower branches grow first
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
                        // Start growing next branches that connect to this one
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
                // Create leaves at branch ends
                tree.branches.forEach(branch => {
                    if (branch.depth >= 1) {
                        const endX = branch.startX + Math.cos(branch.angle) * branch.length;
                        const endY = branch.startY + Math.sin(branch.angle) * branch.length;
                        const numLeaves = 4 + Math.floor(Math.random() * 3);
                        // Distribute leaves in a circle around the branch tip
                        for (let i = 0; i < numLeaves; i++) {
                            const leafAngle = (i / numLeaves) * Math.PI * 2;
                            const radius = 10 + Math.random() * 15;
                            const offsetX = Math.cos(leafAngle) * radius;
                            const offsetY = Math.sin(leafAngle) * radius;
                            tree.leaves.push(new Leaf(endX + offsetX, endY + offsetY));
                        }
                    }
                });
            }
        }
        
        function updateLeaves() {
            tree.leaves.forEach(leaf => {
                if (leaf.growing) {
                    leaf.grow();
                }
            });
        }
        
        function draw() {
            ctx.clearRect(0, 0, canvas.width, canvas.height);
            
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
        }
        
        function animate() {
            if (animationPhase === 'falling') {
                updateSeed();
            } else if (animationPhase === 'growing') {
                updateTree();
            } else if (animationPhase === 'branching') {
                updateBranches();
            } else if (animationPhase === 'leaves') {
                updateLeaves();
            }
            
            draw();
            requestAnimationFrame(animate);
        }
        
        animate();
    </script>
</body>
</html>
