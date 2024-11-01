<script>
	import { onMount } from 'svelte';
	// @ts-ignore
	// @ts-ignore
	// @ts-ignore
	// @ts-ignore
	// @ts-ignore
	import * as d3 from 'd3';
	// @ts-ignore
	import { initializeApp } from 'firebase/app';
	// @ts-ignore
	import { getDatabase, ref, set, onValue, get } from 'firebase/database';

	// @ts-ignore
	const encodeUsername = (text) => text.replaceAll('.', '-');

	const firebaseConfig = {
		apiKey: 'AIzaSyCjXVFc6h-s1UJ5DyUxlmnF3moIG9K6aAc',
		authDomain: 'thainads-speed-dating.firebaseapp.com',
		databaseURL: 'https://thainads-speed-dating-default-rtdb.asia-southeast1.firebasedatabase.app',
		projectId: 'thainads-speed-dating',
		storageBucket: 'thainads-speed-dating.appspot.com',
		messagingSenderId: '753163570473',
		appId: '1:753163570473:web:132c525e1875d70f3a7b38'
	};

	const app = initializeApp(firebaseConfig);
	const db = getDatabase(app);

	let users = {};

	// @ts-ignore
	let connections = [];
	// @ts-ignore
	let graphInstance;
	let gData = { nodes: [], links: [] };
	const boundary = 200;
	const zBoundary = 1000;
	const minNodeSize = 80;
	const maxNodeSize = 160;
	const nodeConnectionCounts = {};

	let isLoaded = false;

	let savedPositions = {}; // Initialize savedPositions
	// @ts-ignore
	let inactivityTimer;
	const inactivityDelay = 3000; // 3 seconds

	onMount(async () => {
		// Dynamically import force-graph in onMount
		const { default: ForceGraph } = await import('force-graph');

		await loadNodePositions();
		getUsers();
		listenForNewConnections();

		renderGraph2D(ForceGraph); // Pass ForceGraph to the function
	});

	function getUsers() {
		const usersRef = ref(db, 'users');

		// @ts-ignore
		onValue(usersRef, (snapshot) => {
			users = snapshot.val() || {};
			if (!isLoaded) {
				isLoaded = true;
				gData = {
					// @ts-ignore
					// @ts-ignore
					nodes: Object.entries(users).map(([key, user]) => {
						const userAvatar = new Image();
						userAvatar.src = user.avatarUrl;
						return {
							id: key,
							// @ts-ignore
							x: savedPositions[key]?.x || Math.random() * boundary * 2 - boundary, // Use saved or random position
							// @ts-ignore
							y: savedPositions[key]?.y || Math.random() * boundary * 2 - boundary,
							// @ts-ignore
							z: savedPositions[key]?.z || Math.random() * zBoundary * 2 - zBoundary,
							size: minNodeSize,
							img: userAvatar
						};
					}),
					links: []
				};

				// Get clientId from URL query parameters
				const urlParams = new URLSearchParams(window.location.search);
				const clientId = urlParams.get('clientId');

				// Check if clientId is the specific client
				if (clientId === 'mainScreen') {
					// Start inactivity timer for this client
					resetInactivityTimer();
					// @ts-ignore
					graphInstance.onEngineStop(resetInactivityTimer);
				}

				Object.keys(users).forEach((username) => {
					// @ts-ignore
					if (username) nodeConnectionCounts[username] = 0;
				});
			} else {
				updateGraph();
			}
		});
	}

	function listenForNewConnections() {
		const connectionsRef = ref(db, 'connections');
		// @ts-ignore
		onValue(connectionsRef, (snapshot) => {
			const newConnection = snapshot.val() || {};
			connections = Object.entries(newConnection).map(([key, value]) => ({
				id: key,
				...value
			}));
			updateGraph();
		});
	}

	// @ts-ignore
	function renderGraph2D(ForceGraph) {
		// @ts-ignore
		if (graphInstance) graphInstance._destructor();

		graphInstance = ForceGraph()(document.getElementById('force-graph'))
			.graphData(gData)
			.nodeAutoColorBy('id')
			.nodeLabel('id')
			.backgroundColor('#000011')
			.linkWidth(0.5)
			.linkColor(() => 'rgba(255, 255, 255, 0.4)')
			// @ts-ignore
			.nodeCanvasObject((node, ctx) => {
				// @ts-ignore
				// const user = users[node.id];
				// const img = new Image();
				// img.src = user.avatarUrl;
				ctx.save();
				ctx.beginPath();
				ctx.arc(node.x, node.y, node.size / 2, 0, 2 * Math.PI, false);
				ctx.clip();
				ctx.drawImage(
					node.img,
					node.x - node.size / 2,
					node.y - node.size / 2,
					node.size,
					node.size
				);
				ctx.restore();
			})
			.enableNodeDrag(false)
			.d3Force(
				'charge',
				d3
					.forceManyBody()
					.strength(-240)
					.distanceMin(40)
					.distanceMax(boundary * 2)
			)
			.d3Force('link', d3.forceLink().distance(300).strength(1))
			.d3Force('center', d3.forceCenter(boundary / 2, boundary / 2))
			// @ts-ignore
			.d3Force(
				'collision',
				// @ts-ignore
				d3.forceCollide((node) => node.size / 2 + 10)
			);
	}

	// @ts-ignore
	function mapDegreeToSize(degree, minDegree, maxDegree, minSize, maxSize) {
		return minSize + ((maxSize - minSize) * (degree - minDegree)) / (maxDegree - minDegree);
	}

	// @ts-ignore
	function calculateNodeSize(connections) {
		const minDegree = 0;
		const maxDegree = 10;
		return mapDegreeToSize(connections, minDegree, maxDegree, minNodeSize, maxNodeSize);
	}

	function updateConnectionCounts() {
		Object.entries(users).forEach(([key, user]) => {
			// @ts-ignore
			nodeConnectionCounts[key] = user?.connections ? Object.keys(user?.connections).length : 0;
		});
		let totalConnections = Object.values(nodeConnectionCounts).reduce((a, b) => a + b, 0);
		let averageConnections = Math.round(totalConnections / Object.keys(users).length);
		let maxConnections = Math.max(...Object.values(nodeConnectionCounts));
		let minConnections = Math.min(...Object.values(nodeConnectionCounts));
		console.log(
			`Total users: ${Object.keys(users).length}, \nTotal connections: ${totalConnections}, \nAverage connections: ${averageConnections}, \nMax connections: ${maxConnections}, \nMin connections: ${minConnections}`
		);
	}

	function updateNodeSizes() {
		gData.nodes.forEach((node) => {
			// @ts-ignore
			node.size = calculateNodeSize(nodeConnectionCounts[node.id]);
		});
	}

	function updateGraph() {
		// @ts-ignore
		const links = connections.map((connection) => ({
			source: encodeUsername(connection.user1),
			target: encodeUsername(connection.user2)
		}));

		// @ts-ignore
		gData.links = [...links];
		updateConnectionCounts();
		updateNodeSizes();
		// @ts-ignore
		graphInstance.graphData(gData);
	}

	function saveNodePositions() {
		const positionsRef = ref(db, 'nodePositions');
		const positions = {};
		gData.nodes.forEach((node) => {
			// @ts-ignore
			positions[node.id] = { x: node.x, y: node.y, z: node.z };
		});
		// @ts-ignore
		set(positionsRef, positions);
	}

	// @ts-ignore
	// @ts-ignore
	// @ts-ignore
	async function loadNodePositions() {
		const positionsRef = ref(db, 'nodePositions');
		const snapshot = await get(positionsRef);
		savedPositions = snapshot.val() || {};
	}

	// @ts-ignore
	function resetInactivityTimer() {
		// @ts-ignore
		clearTimeout(inactivityTimer);
		inactivityTimer = setTimeout(saveNodePositions, inactivityDelay);
	}
</script>

<!-- Test commit -->

<div id="graph-container">
	<div id="force-graph" />
</div>

<style>
	#graph-container {
		position: relative;
		width: 100%;
		height: 100%;
	}
</style>
