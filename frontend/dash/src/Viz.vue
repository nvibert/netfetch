<template>
    <div class="network-policy-visualization" ref="vizContainer"></div>
  </template>  
  
  <script>
  import * as d3 from 'd3';

  // Define the custom cluster force
  function forceCluster(centers, nodeCounts) {
  let nodes;
  let strength = 0.1;

  function force(alpha) {
    const l = alpha * strength;
    for (const node of nodes) {
      const center = centers[node.cluster];
      const count = nodeCounts[node.cluster] || 1;
      node.vx -= (node.x - center.x) * l / count;
      node.vy -= (node.y - center.y) * l / count;
    }
  }

  force.initialize = function(_) {
    nodes = _;
  };

  force.strength = function(_) {
    return arguments.length ? (strength = +_, force) : strength;
  };

  return force;
}

  export default {
  name: 'NetworkPolicyVisualization',
  props: {
    policies: Array,
    clusterData: {
      type: Array,
      default: () => [],
    },
    visualizationType: {
      type: String,
      default: 'namespace' // Possible values: 'namespace', 'cluster'
    },
  },
  mounted() {
    try {
      this.createNetworkMap();
    } catch (error) {
      console.error("An error occurred during mounted hook:", error);
    }
  },
  methods: {
    createNetworkMap() {
    let data;
    if (this.visualizationType === 'cluster') {
      data = this.clusterData.reduce((acc, item) => {
        return item.policies && Array.isArray(item.policies) ? [...acc, ...item.policies] : acc;
      }, []);
    } else {
      data = this.policies;
    }

    if (!Array.isArray(data)) {
      console.error('Visualization data is undefined or not an array');
      return;
    }

    const color = d3.scaleOrdinal(d3.schemeCategory10);
    const width = this.$refs.vizContainer.clientWidth;
    const height = this.$refs.vizContainer.clientHeight;
    const nodes = [];
    const links = [];
    const namespaceClusterMap = new Map();

    // Group policies by namespace and prepare nodes and links
    data.forEach(policy => {
      if (!namespaceClusterMap.has(policy.namespace)) {
        namespaceClusterMap.set(policy.namespace, policy.namespace);
      }
      const namespaceCluster = namespaceClusterMap.get(policy.namespace);

      const policyNode = { id: policy.name, type: 'policy', cluster: namespaceCluster };
      nodes.push(policyNode);

      policy.targetPods.forEach(podName => {
        const podNode = { id: podName, type: 'pod', cluster: namespaceCluster };
        if (!nodes.some(n => n.id === podName)) {
          nodes.push(podNode);
        }
        links.push({ source: policy.name, target: podName });
      });
    });

    const clusterNodeCounts = nodes.reduce((acc, node) => {
        acc[node.cluster] = (acc[node.cluster] || 5) + 1;
        return acc;
    }, {});

    const namespaceSizes = data.reduce((sizes, policy) => {
      sizes[policy.namespace] = (sizes[policy.namespace] || 0) + policy.targetPods.length;
      return sizes;
    }, {});

    const dynamicSizes = calculateDynamicSectionSizes(namespaceSizes);
    const dynamicPositions = calculateDynamicSectionPositions(dynamicSizes, width, height);

    console.log('Dynamic Sizes:', dynamicSizes);
    console.log('Dynamic Positions:', dynamicPositions);

    // Now, you can use dynamicPositions to set your cluster centers
    const clusterCenters = {};
    Object.keys(dynamicPositions).forEach(namespace => {
      clusterCenters[namespace] = dynamicPositions[namespace];
    });

    // Tooltip for full text display
    const tooltip = d3.select('body').append('div')
          .attr('class', 'tooltip')
          .style('visibility', 'hidden')
          .style('position', 'absolute')
          .style('background', 'white')
          .style('border', '1px solid black')
          .style('padding', '5px')
          .style('pointer-events', 'none');

    // Calculate cluster centers dynamically.
    const gridSizeMultiplier = 1.2;
    const gridSize = Math.ceil(Math.sqrt(namespaceClusterMap.size) * gridSizeMultiplier);
    const sectionWidth = width / gridSize;
    const sectionHeight = height / gridSize;

    const namespaces = Array.from(namespaceClusterMap.keys());
    namespaces.forEach((namespace, index) => {
      const row = Math.floor(index / gridSize);
      const col = index % gridSize;
      const x = col * sectionWidth + sectionWidth / 2;
      const y = row * sectionHeight + sectionHeight / 2;
      clusterCenters[namespaceClusterMap.get(namespace)] = { x, y };
    });

    function calculateDynamicSectionSizes(namespaceSizes) {
    const dynamicSizes = {};
    const padding = 50;
    for (const namespace in namespaceSizes) {
      const size = namespaceSizes[namespace];
      // Increase the arbitrary scaling factor, add padding, and ensure a minimum size
      dynamicSizes[namespace] = {
        width: Math.max(Math.sqrt(size) * 20 + padding, 200), // Increase the minimum size if necessary
        height: Math.max(Math.sqrt(size) * 20 + padding, 200)
      };
    }
    return dynamicSizes;
  }

  function calculateDynamicSectionPositions(dynamicSizes, width, height) {
    const positions = {};
    let i = 0;
    for (const namespace in dynamicSizes) {
      // Use namespace directly instead of index
      const x = (i % 10) * (width / 10) + dynamicSizes[namespace].width / 2;
      const y = Math.floor(i / 10) * (height / 10) + dynamicSizes[namespace].height / 2;
      positions[namespace] = { x, y };
      i++;
    }
    return positions;
  }

  function applyContainmentForce(nodes, clusterCenters, dynamicSizes, clusterNodeCounts) {
    // Iterate over each node and apply containment logic
    nodes.forEach(node => {
      const center = clusterCenters[node.cluster];
      const size = dynamicSizes[node.cluster];
      const count = clusterNodeCounts[node.cluster] || 1;

      // Calculate the radius based on the number of nodes and the size of the section
      const radius = Math.sqrt(count) * (size.width / 2);

      // Calculate the distance from the node to the cluster center
      const dx = node.x - center.x;
      const dy = node.y - center.y;
      const distance = Math.sqrt(dx * dx + dy * dy);

      // If the node is outside the cluster radius, bring it back to the boundary
      if (distance > radius) {
        const angle = Math.atan2(dy, dx);
        node.x = center.x + Math.cos(angle) * radius;
        node.y = center.y + Math.sin(angle) * radius;
      }
    });
  }

    // Assign initial positions
    nodes.forEach(node => {
      const center = clusterCenters[node.cluster];
      node.x = center ? center.x : width / 2;
      node.y = center ? center.y : height / 2;
    });

    console.log('Nodes after initial position:', nodes);
    console.log('Cluster Centers:', clusterCenters);

    d3.select(this.$refs.vizContainer).selectAll('svg').remove();

    // Setup zoom behavior
    const zoom = d3.zoom()
        .scaleExtent([0.1, 10])
        .translateExtent([[-width, -height], [10 * width, 10 * height]]) // This should cover the zoomed area
        .on('zoom', (event) => {
            containerGroup.attr('transform', event.transform);
        });

    // Create SVG element
    const svg = d3.select(this.$refs.vizContainer)
      .append('svg')
      .attr('width', '100%')
      .attr('height', '100%')
      .attr('viewBox', `0 0 ${width} ${height}`)

    svg.call(zoom);

    const containerGroup = svg.append('g');
    const sizeFactor = 3;

    if (this.visualizationType === 'cluster') {
      const initialTransform = d3.zoomIdentity.translate(width / 8, height / 2.6).scale(0.3);
      svg.call(zoom.transform, initialTransform);
    }

    if (this.visualizationType === 'namespace') {
      const initialTransform = d3.zoomIdentity.translate(width / 8, height / 4).scale(0.6);
      svg.call(zoom.transform, initialTransform);
    }

    const adjustedClusterCenters = {};
    Object.keys(clusterCenters).forEach((cluster) => {
      const namespaceSize = namespaceSizes[cluster];
      const offsetX = (namespaceSize * sizeFactor) / 2;
      const offsetY = (namespaceSize * sizeFactor) / 2;
      adjustedClusterCenters[cluster] = {
        x: clusterCenters[cluster].x + offsetX,
        y: clusterCenters[cluster].y + offsetY
      };
    });

    const nodeRadius = 10;
    const collisionRadius = d => {
      const clusterSize = clusterNodeCounts[d.cluster] || 1;
      return nodeRadius + (clusterSize * 0.6);
    };

    const linkDistance = (d) => {
    const clusterSize = clusterNodeCounts[d.source.cluster] || 1;
    const baseDistance = 30;
    const additionalDistancePerNode = 1.5;

    return baseDistance + (clusterSize * additionalDistancePerNode);
  };

  const ticked = () => {
  // Apply containment force to nodes
  applyContainmentForce(nodes, clusterCenters, dynamicSizes, clusterNodeCounts);

  link
    .attr('x1', d => {
      const dx = d.target.x - d.source.x;
      const dy = d.target.y - d.source.y;
      const distance = Math.sqrt(dx * dx + dy * dy);
      const sourceRadius = d.source.type === 'pod' ? 5 : 7.2;
      return d.source.x + (dx * sourceRadius) / distance;
    })
    .attr('y1', d => {
      const dx = d.target.x - d.source.x;
      const dy = d.target.y - d.source.y;
      const distance = Math.sqrt(dx * dx + dy * dy);
      const sourceRadius = d.source.type === 'pod' ? 5 : 7.2;
      return d.source.y + (dy * sourceRadius) / distance;
    })
    .attr('x2', d => {
      const dx = d.source.x - d.target.x;
      const dy = d.source.y - d.target.y;
      const distance = Math.sqrt(dx * dx + dy * dy);
      const targetRadius = d.target.type === 'pod' ? 5 : 7.2;
      return d.target.x + (dx * targetRadius) / distance;
    })
    .attr('y2', d => {
      const dx = d.source.x - d.target.x;
      const dy = d.source.y - d.target.y;
      const distance = Math.sqrt(dx * dx + dy * dy);
      const targetRadius = d.target.type === 'pod' ? 5 : 7.2;
      return d.target.y + (dy * targetRadius) / distance;
    });
      
    node
      .attr('cx', d => d.x)
      .attr('cy', d => d.y);

    labels
      .attr('x', d => d.x)
      .attr('y', d => d.y);
  }

    // Create the simulation with appropriate forces
    const simulation = d3.forceSimulation(nodes)
      .force('link', d3.forceLink(links).id(d => d.id).distance(linkDistance))
      .force('charge', d3.forceManyBody().strength(-100))
      .force('collide', d3.forceCollide().radius(d => collisionRadius(d) + 35).iterations(6))
      .force('x', d3.forceX(d => clusterCenters[d.cluster].x).strength(0.1))
      .force('y', d3.forceY(d => clusterCenters[d.cluster].y).strength(0.1))
      .force('cluster', forceCluster(clusterCenters, clusterNodeCounts).strength(0.2))
      .on('tick', ticked);

      const node = containerGroup.append('g')
        .selectAll('circle')
        .data(nodes)
        .join('circle')
        .attr('r', d => {
          if (d.type === 'pod') {
            return 5; // Radius for pods
          } else if (d.type === 'policy') {
            return 7.2; // Larger radius for policies
          }
          return 5; // Default radius
        })
        .attr('fill', d => {
          if (d.type === 'pod') {
            return 'coral'; // Pods are green
          } else if (d.type === 'policy') {
            return 'teal'; // Policies are blue
          }
          return color(d.cluster); // Default color
        })
        .attr('stroke', 'grey')
        .attr('stroke-width', 1.5)
        .call(drag(simulation));

    // Legends
    const legendGroup = svg.append('g')
      .attr('class', 'legend')
      .attr('transform', 'translate(-20, 10)');

    legendGroup.append('circle')
      .attr('r', 5)
      .attr('fill', 'coral')
      .attr('cx', 0)
      .attr('cy', 0);

    legendGroup.append('text')
      .attr('x', 20)
      .attr('y', 5)
      .text('Pod');

    legendGroup.append('circle')
      .attr('r', 5)
      .attr('fill', 'teal')
      .attr('cx', 0)
      .attr('cy', 20);

    legendGroup.append('text')
      .attr('x', 20)
      .attr('y', 25)
      .text('Policy');

    // Create links and nodes inside the containerGroup
    const link = containerGroup.append('g')
          .attr('stroke', 'grey')
          .attr('stroke-opacity', 1)
          .selectAll('line')
          .data(links)
          .join('line')
          .attr('stroke-width', d => Math.sqrt(d.value));

    // Add labels with hover functionality to show full text
    const labels = containerGroup.append('g')
        .attr('class', 'labels')
        .selectAll('text')
        .data(nodes)
        .enter().append('text')
        .attr('dx', 12)
        .attr('dy', '.35em')
        .text(d => {
            // Display full name for policy and truncate for pod
            return d.type === 'policy' ? d.id : (d.id.substring(0, 10) + (d.id.length > 10 ? '...' : ''));
        })
        .on('mouseover', (event, d) => {
            tooltip.html(d.id)
                .style('visibility', 'visible')
                .style('left', (event.pageX + 10) + 'px')
                .style('top', (event.pageY - 10) + 'px');
        })
        .on('mouseout', () => {
            tooltip.style('visibility', 'hidden');
        });

        simulation.on('tick', () => {
          // Keep nodes within the circular boundaries
          nodes.forEach(node => {
            const center = clusterCenters[node.cluster];
            const radius = Math.sqrt(clusterNodeCounts[node.cluster] || 1) * (dynamicSizes[node.cluster].width / 2); // Calculate radius
            const dx = node.x - center.x;
            const dy = node.y - center.y;
            const distance = Math.sqrt(dx * dx + dy * dy);

            if (distance > radius) {
              // If node is outside the radius, bring it back to the edge of the circle
              node.x = center.x + (dx / distance) * radius;
              node.y = center.y + (dy / distance) * radius;
            }
          });

          // Update link positions, checking for NaN values
          link
            .attr('x1', d => {
              const dx = d.target.x - d.source.x;
              const dy = d.target.y - d.source.y;
              const distance = Math.sqrt(dx * dx + dy * dy);
              const sourceRadius = d.source.type === 'pod' ? 5 : 7.2;
              return distance ? d.source.x + (dx * sourceRadius) / distance : d.source.x;
            })
            .attr('y1', d => {
              const dx = d.target.x - d.source.x;
              const dy = d.target.y - d.source.y;
              const distance = Math.sqrt(dx * dx + dy * dy);
              const sourceRadius = d.source.type === 'pod' ? 5 : 7.2;
              return distance ? d.source.y + (dy * sourceRadius) / distance : d.source.y;
            })
            .attr('x2', d => {
              const dx = d.source.x - d.target.x;
              const dy = d.source.y - d.target.y;
              const distance = Math.sqrt(dx * dx + dy * dy);
              const targetRadius = d.target.type === 'pod' ? 5 : 7.2;
              return distance ? d.target.x + (dx * targetRadius) / distance : d.target.x;
            })
            .attr('y2', d => {
              const dx = d.source.x - d.target.x;
              const dy = d.source.y - d.target.y;
              const distance = Math.sqrt(dx * dx + dy * dy);
              const targetRadius = d.target.type === 'pod' ? 5 : 7.2;
              return distance ? d.target.y + (dy * targetRadius) / distance : d.target.y;
            });

          // Update node positions, checking for NaN values
          node
            .attr('cx', d => isNaN(d.x) ? 0 : d.x)
            .attr('cy', d => isNaN(d.y) ? 0 : d.y);

          // Update label positions, checking for NaN values
          labels
            .attr('x', d => isNaN(d.x) ? 0 : d.x)
            .attr('y', d => isNaN(d.y) ? 0 : d.y);
        });
        
        svg.append('text')
          .attr('x', width - 200)
          .attr('y', height - 10)
          .text('Drag to move, scroll to zoom.')
          .style('font-size', '18px')
          .style('fill', 'black');
    
    // Drag functionality
    function drag(simulation) {
      function dragstarted(event) {
        if (!event.active) simulation.alphaTarget(0.1).restart();
        event.subject.fx = event.subject.x;
        event.subject.fy = event.subject.y;
      }

      function dragged(event) {
        event.subject.fx = event.x;
        event.subject.fy = event.y;
      }

      function dragended(event) {
        if (!event.active) simulation.alphaTarget(0);
        event.subject.fx = null;
        event.subject.fy = null;
      }

      return d3.drag()
          .on('start', dragstarted)
          .on('drag', dragged)
          .on('end', dragended);
          }
      },
    },
    watch: {
        policies(newVal) {
            console.log('Policies data received:', newVal);
            this.createNetworkMap();
        }
    }
};
</script>
  
<style scoped>
.network-policy-visualization svg {
  border: 1px solid #87CEEB;
  border-radius: 5px;
  background-color: #f9f9f9;
}

.network-policy-visualization circle {
  stroke: #87CEEB;
  stroke-width: 1.5px;
}

.network-policy-visualization text {
  font-size: 10px;
  pointer-events: none;
}

.network-policy-visualization {
  width: 100%;
  padding: 20px;
  height: 600px;
}

.tooltip {
  position: absolute;
  visibility: hidden;
  background: white;
  border: 1px solid black;
  padding: 5px;
  pointer-events: none;
  z-index: 10;
}

</style>
  