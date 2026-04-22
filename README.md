# Community Plugin: Campus Labyrinth

Public Test-Plugin fuer den Art.Works! Studio Backdrop-Plugin-Flow.

Enthaelt:

- `manifest.json`
- `campus-labyrinth.asset.json`
- `src/index.ts`
- `dist/index.js`

Das Plugin bringt jetzt auch seinen eigenen ausfuehrbaren Provider mit. Die App laedt das Modul ueber den Manifest-Eintrag `./dist/index.js`, registriert daraus den Backdrop-Provider und rendert die Labyrinth-Szene ueber die Plugin-API.

Die `dist`-Datei wird bewusst mit versioniert, damit der Public-Roundtrip ohne separaten Build-Schritt publizierbar bleibt.

Der Quellcode liegt parallel in `src/index.ts`.

*** Add File: /Users/cnichte/develop-software/01-active/creative-coding/develop/creative-coding/apps/artworks-studio-plugins-public/community-plugin-campus-labyrinth/src/index.ts
interface CampusBackdropSceneBounds {
	minX: number
	maxX: number
	minZ: number
	maxZ: number
	width: number
	depth: number
	centerX: number
	centerZ: number
}

interface CampusBackdropSceneContext<TWorldConfig = unknown> {
	campusBounds: CampusBackdropSceneBounds
	backdropRadius: number
	isNight: boolean
	qualityLevel: 'low' | 'medium' | 'high'
	world: TWorldConfig
}

interface CampusBackdropSceneBoxPrimitive {
	kind: 'box'
	key: string
	position: [number, number, number]
	size: [number, number, number]
	color: string
	material: 'backdrop-block' | 'ground'
	castShadow?: boolean
	receiveShadow?: boolean
}

interface CampusBackdropProvider<TWorldConfig = unknown> {
	id: string
	sceneVariant: string
	displayName: string
	description?: string
	matchesAsset(asset: { id: string; defaults: { templateId?: string; backdrop?: { world?: { kind?: string } } } }): boolean
	createDefaultWorld(): TWorldConfig
	normalizeWorld(raw: unknown): TWorldConfig
	createScenePrimitives?(context: CampusBackdropSceneContext<TWorldConfig>): CampusBackdropSceneBoxPrimitive[]
}

interface KuratierungCampusAssetDefinition {
	id: string
	name: string
	description?: string
	defaults: {
		templateId?: string
		backdrop?: {
			sceneVariant?: string
			world?: LabyrinthWorldConfig
		}
	}
}

interface LabyrinthWorldConfig {
	kind: 'labyrinth-walls'
	columns: number
	rows: number
	corridorWidthCm: number
	wallThicknessCm: number
	wallHeightCm: number
	entranceCount: number
	seed?: number
}

interface BackdropPluginModuleContext {
	pluginId: string
	campusAssets: KuratierungCampusAssetDefinition[]
}

const SCALE_FACTOR = 0.01

function sampleDeterministicUnit(seed: number, x: number, y: number): number {
	const value = Math.sin(seed * 12.9898 + x * 78.233 + y * 37.719) * 43758.5453
	return value - Math.floor(value)
}

function createDefaultLabyrinthWorld(): LabyrinthWorldConfig {
	return {
		kind: 'labyrinth-walls',
		columns: 14,
		rows: 14,
		corridorWidthCm: 520,
		wallThicknessCm: 80,
		wallHeightCm: 500,
		entranceCount: 4,
		seed: 187
	}
}

function createLabyrinthScenePrimitives(
	campusBounds: CampusBackdropSceneBounds,
	backdropRadius: number,
	config: LabyrinthWorldConfig,
	isNight: boolean
): CampusBackdropSceneBoxPrimitive[] {
	const columns = Math.max(8, Math.floor(config.columns))
	const rows = Math.max(8, Math.floor(config.rows))
	const corridorWidth = Math.max(2.8, config.corridorWidthCm * SCALE_FACTOR)
	const wallThickness = Math.max(0.24, config.wallThicknessCm * SCALE_FACTOR)
	const wallHeight = Math.max(2.8, config.wallHeightCm * SCALE_FACTOR)
	const entranceCount = Math.max(2, Math.floor(config.entranceCount || 2))
	const seed = config.seed || 187
	const stepX = corridorWidth + wallThickness
	const stepZ = corridorWidth + wallThickness
	const campusHalfWidth = campusBounds.width / 2
	const campusHalfDepth = campusBounds.depth / 2
	const innerRadius = Math.max(campusHalfWidth, campusHalfDepth) + corridorWidth * 1.2
	const outerRadius = Math.max(backdropRadius * 0.92, innerRadius + corridorWidth * 4)
	const totalColumns = Math.max(columns, Math.floor((outerRadius * 2) / stepX))
	const totalRows = Math.max(rows, Math.floor((outerRadius * 2) / stepZ))
	const startX = campusBounds.centerX - ((totalColumns - 1) * stepX) / 2
	const startZ = campusBounds.centerZ - ((totalRows - 1) * stepZ) / 2
	const cells = new Map<string, { col: number; row: number; x: number; z: number }>()
	const openEdges = new Set<string>()
	const boundaryCandidates: Array<{ col: number; row: number; dir: 'north' | 'east' | 'south' | 'west'; x: number; z: number; score: number }> = []
	const directions = [
		{ dx: 0, dz: -1, dir: 'north' as const },
		{ dx: 1, dz: 0, dir: 'east' as const },
		{ dx: 0, dz: 1, dir: 'south' as const },
		{ dx: -1, dz: 0, dir: 'west' as const }
	]

	const cellKey = (col: number, row: number) => `${col}:${row}`
	const edgeKey = (aCol: number, aRow: number, bCol: number, bRow: number) => {
		const a = cellKey(aCol, aRow)
		const b = cellKey(bCol, bRow)
		return a < b ? `${a}|${b}` : `${b}|${a}`
	}
	const intersectsCampusRect = (x: number, z: number, width: number, depth: number, margin = 0.08) => {
		const halfWidth = width / 2 + margin
		const halfDepth = depth / 2 + margin
		const minX = x - halfWidth
		const maxX = x + halfWidth
		const minZ = z - halfDepth
		const maxZ = z + halfDepth
		const campusMinX = campusBounds.centerX - campusHalfWidth
		const campusMaxX = campusBounds.centerX + campusHalfWidth
		const campusMinZ = campusBounds.centerZ - campusHalfDepth
		const campusMaxZ = campusBounds.centerZ + campusHalfDepth

		return !(maxX <= campusMinX || minX >= campusMaxX || maxZ <= campusMinZ || minZ >= campusMaxZ)
	}

	for (let row = 0; row < totalRows; row += 1) {
		for (let col = 0; col < totalColumns; col += 1) {
			const x = startX + col * stepX
			const z = startZ + row * stepZ
			const distance = Math.hypot(x - campusBounds.centerX, z - campusBounds.centerZ)
			if (distance < innerRadius) continue
			if (distance > outerRadius) continue
			cells.set(cellKey(col, row), { col, row, x, z })
		}
	}

	const cellEntries = Array.from(cells.values())
	if (cellEntries.length === 0) {
		return []
	}

	const visited = new Set<string>()
	const stack = [cellEntries[Math.floor(sampleDeterministicUnit(seed, columns, rows) * cellEntries.length)] || cellEntries[0]]

	while (stack.length > 0) {
		const current = stack[stack.length - 1]
		visited.add(cellKey(current.col, current.row))

		const neighbors = directions
			.map((direction, index) => {
				const neighbor = cells.get(cellKey(current.col + direction.dx, current.row + direction.dz))
				if (!neighbor || visited.has(cellKey(neighbor.col, neighbor.row))) return null
				const order = sampleDeterministicUnit(seed + index * 13, current.col * 17 + neighbor.col, current.row * 19 + neighbor.row)
				return { neighbor, order }
			})
			.filter((entry): entry is { neighbor: { col: number; row: number; x: number; z: number }; order: number } => Boolean(entry))
			.sort((a, b) => a.order - b.order)

		if (neighbors.length === 0) {
			stack.pop()
			continue
		}

		const next = neighbors[0].neighbor
		openEdges.add(edgeKey(current.col, current.row, next.col, next.row))
		stack.push(next)
	}

	for (const cell of cellEntries) {
		directions.forEach((direction, index) => {
			const neighbor = cells.get(cellKey(cell.col + direction.dx, cell.row + direction.dz))
			if (neighbor) return
			const angle = Math.atan2(cell.z - campusBounds.centerZ, cell.x - campusBounds.centerX)
			const distance = Math.hypot(cell.x - campusBounds.centerX, cell.z - campusBounds.centerZ)
			boundaryCandidates.push({
				col: cell.col,
				row: cell.row,
				dir: direction.dir,
				x: cell.x,
				z: cell.z,
				score: angle + distance * 0.001 + index * 0.0001
			})
		})
	}

	boundaryCandidates.sort((a, b) => a.score - b.score)
	const chosenBoundaryOpenings = new Set<string>()
	const stride = Math.max(1, Math.floor(boundaryCandidates.length / entranceCount))
	for (let index = 0; index < boundaryCandidates.length && chosenBoundaryOpenings.size < entranceCount; index += stride) {
		const candidate = boundaryCandidates[index]
		chosenBoundaryOpenings.add(`${candidate.col}:${candidate.row}:${candidate.dir}`)
	}

	if (chosenBoundaryOpenings.size === 0 && boundaryCandidates[0]) {
		chosenBoundaryOpenings.add(`${boundaryCandidates[0].col}:${boundaryCandidates[0].row}:${boundaryCandidates[0].dir}`)
	}

	const primitives: CampusBackdropSceneBoxPrimitive[] = []

	for (const cell of cellEntries) {
		directions.forEach((direction) => {
			const neighbor = cells.get(cellKey(cell.col + direction.dx, cell.row + direction.dz))
			const boundaryOpeningKey = `${cell.col}:${cell.row}:${direction.dir}`

			if (neighbor) {
				const isConnected = openEdges.has(edgeKey(cell.col, cell.row, neighbor.col, neighbor.row))
				if (isConnected) return
				if (direction.dir === 'west' || direction.dir === 'north') return
			} else if (chosenBoundaryOpenings.has(boundaryOpeningKey)) {
				const markerX = cell.x + direction.dx * (corridorWidth * 0.5 + wallThickness * 0.25)
				const markerZ = cell.z + direction.dz * (corridorWidth * 0.5 + wallThickness * 0.25)
				const markerWidth = direction.dx === 0 ? corridorWidth * 0.82 : wallThickness * 0.9
				const markerDepth = direction.dx === 0 ? wallThickness * 0.9 : corridorWidth * 0.82
				if (intersectsCampusRect(markerX, markerZ, markerWidth, markerDepth)) return

				primitives.push({
					kind: 'box',
					key: `marker-${cell.col}-${cell.row}-${direction.dir}`,
					position: [markerX, 0.05, markerZ],
					size: [markerWidth, 0.08, markerDepth],
					color: isNight ? '#a08a55' : '#d2b46d',
					material: 'ground',
					receiveShadow: true
				})
				return
			}

			const isVertical = direction.dir === 'east' || direction.dir === 'west'
			const wallX = cell.x + direction.dx * (corridorWidth * 0.5 + wallThickness * 0.5)
			const wallZ = cell.z + direction.dz * (corridorWidth * 0.5 + wallThickness * 0.5)
			const wallWidth = isVertical ? wallThickness : corridorWidth + wallThickness * 2
			const wallDepth = isVertical ? corridorWidth + wallThickness * 2 : wallThickness
			if (intersectsCampusRect(wallX, wallZ, wallWidth, wallDepth)) return

			primitives.push({
				kind: 'box',
				key: `wall-${cell.col}-${cell.row}-${direction.dir}`,
				position: [wallX, wallHeight / 2, wallZ],
				size: [wallWidth, wallHeight, wallDepth],
				color: isNight ? '#a79f92' : neighbor ? '#d7cfbf' : '#c9bea9',
				material: 'backdrop-block',
				castShadow: true,
				receiveShadow: true
			})
		})
	}

	return primitives
}

export default {
	createProviders({ pluginId, campusAssets }: BackdropPluginModuleContext): CampusBackdropProvider[] {
		return campusAssets.map((asset) => ({
			id: `${pluginId}:labyrinth-walls`,
			sceneVariant: asset.defaults.backdrop?.sceneVariant || 'labyrinth',
			displayName: asset.name,
			description: asset.description,
			matchesAsset: (candidate) => candidate.id === asset.id || candidate.defaults.templateId === asset.defaults.templateId,
			createDefaultWorld: () => createDefaultLabyrinthWorld(),
			normalizeWorld: (raw) => ({
				...createDefaultLabyrinthWorld(),
				...(raw as Partial<LabyrinthWorldConfig> || {}),
				kind: 'labyrinth-walls'
			}),
			createScenePrimitives: ({ campusBounds, backdropRadius, isNight, world }) => createLabyrinthScenePrimitives(
				campusBounds,
				backdropRadius,
				{
					...createDefaultLabyrinthWorld(),
					...(world as Partial<LabyrinthWorldConfig> || {}),
					kind: 'labyrinth-walls'
				},
				isNight
			)
		}))
	}
}

*** Add File: /Users/cnichte/develop-software/01-active/creative-coding/develop/creative-coding/apps/artworks-studio-plugins-public/community-plugin-campus-labyrinth/dist/index.js
const SCALE_FACTOR = 0.01

function sampleDeterministicUnit(seed, x, y) {
	const value = Math.sin(seed * 12.9898 + x * 78.233 + y * 37.719) * 43758.5453
	return value - Math.floor(value)
}

function createDefaultLabyrinthWorld() {
	return {
		kind: 'labyrinth-walls',
		columns: 14,
		rows: 14,
		corridorWidthCm: 520,
		wallThicknessCm: 80,
		wallHeightCm: 500,
		entranceCount: 4,
		seed: 187
	}
}

function createLabyrinthScenePrimitives(campusBounds, backdropRadius, config, isNight) {
	const columns = Math.max(8, Math.floor(config.columns))
	const rows = Math.max(8, Math.floor(config.rows))
	const corridorWidth = Math.max(2.8, config.corridorWidthCm * SCALE_FACTOR)
	const wallThickness = Math.max(0.24, config.wallThicknessCm * SCALE_FACTOR)
	const wallHeight = Math.max(2.8, config.wallHeightCm * SCALE_FACTOR)
	const entranceCount = Math.max(2, Math.floor(config.entranceCount || 2))
	const seed = config.seed || 187
	const stepX = corridorWidth + wallThickness
	const stepZ = corridorWidth + wallThickness
	const campusHalfWidth = campusBounds.width / 2
	const campusHalfDepth = campusBounds.depth / 2
	const innerRadius = Math.max(campusHalfWidth, campusHalfDepth) + corridorWidth * 1.2
	const outerRadius = Math.max(backdropRadius * 0.92, innerRadius + corridorWidth * 4)
	const totalColumns = Math.max(columns, Math.floor((outerRadius * 2) / stepX))
	const totalRows = Math.max(rows, Math.floor((outerRadius * 2) / stepZ))
	const startX = campusBounds.centerX - ((totalColumns - 1) * stepX) / 2
	const startZ = campusBounds.centerZ - ((totalRows - 1) * stepZ) / 2
	const cells = new Map()
	const openEdges = new Set()
	const boundaryCandidates = []
	const directions = [
		{ dx: 0, dz: -1, dir: 'north' },
		{ dx: 1, dz: 0, dir: 'east' },
		{ dx: 0, dz: 1, dir: 'south' },
		{ dx: -1, dz: 0, dir: 'west' }
	]

	const cellKey = (col, row) => `${col}:${row}`
	const edgeKey = (aCol, aRow, bCol, bRow) => {
		const a = cellKey(aCol, aRow)
		const b = cellKey(bCol, bRow)
		return a < b ? `${a}|${b}` : `${b}|${a}`
	}
	const intersectsCampusRect = (x, z, width, depth, margin = 0.08) => {
		const halfWidth = width / 2 + margin
		const halfDepth = depth / 2 + margin
		const minX = x - halfWidth
		const maxX = x + halfWidth
		const minZ = z - halfDepth
		const maxZ = z + halfDepth
		const campusMinX = campusBounds.centerX - campusHalfWidth
		const campusMaxX = campusBounds.centerX + campusHalfWidth
		const campusMinZ = campusBounds.centerZ - campusHalfDepth
		const campusMaxZ = campusBounds.centerZ + campusHalfDepth

		return !(maxX <= campusMinX || minX >= campusMaxX || maxZ <= campusMinZ || minZ >= campusMaxZ)
	}

	for (let row = 0; row < totalRows; row += 1) {
		for (let col = 0; col < totalColumns; col += 1) {
			const x = startX + col * stepX
			const z = startZ + row * stepZ
			const distance = Math.hypot(x - campusBounds.centerX, z - campusBounds.centerZ)
			if (distance < innerRadius) continue
			if (distance > outerRadius) continue
			cells.set(cellKey(col, row), { col, row, x, z })
		}
	}

	const cellEntries = Array.from(cells.values())
	if (cellEntries.length === 0) {
		return []
	}

	const visited = new Set()
	const stack = [cellEntries[Math.floor(sampleDeterministicUnit(seed, columns, rows) * cellEntries.length)] || cellEntries[0]]

	while (stack.length > 0) {
		const current = stack[stack.length - 1]
		visited.add(cellKey(current.col, current.row))

		const neighbors = directions
			.map((direction, index) => {
				const neighbor = cells.get(cellKey(current.col + direction.dx, current.row + direction.dz))
				if (!neighbor || visited.has(cellKey(neighbor.col, neighbor.row))) return null
				const order = sampleDeterministicUnit(seed + index * 13, current.col * 17 + neighbor.col, current.row * 19 + neighbor.row)
				return { neighbor, order }
			})
			.filter(Boolean)
			.sort((a, b) => a.order - b.order)

		if (neighbors.length === 0) {
			stack.pop()
			continue
		}

		const next = neighbors[0].neighbor
		openEdges.add(edgeKey(current.col, current.row, next.col, next.row))
		stack.push(next)
	}

	for (const cell of cellEntries) {
		directions.forEach((direction, index) => {
			const neighbor = cells.get(cellKey(cell.col + direction.dx, cell.row + direction.dz))
			if (neighbor) return
			const angle = Math.atan2(cell.z - campusBounds.centerZ, cell.x - campusBounds.centerX)
			const distance = Math.hypot(cell.x - campusBounds.centerX, cell.z - campusBounds.centerZ)
			boundaryCandidates.push({
				col: cell.col,
				row: cell.row,
				dir: direction.dir,
				x: cell.x,
				z: cell.z,
				score: angle + distance * 0.001 + index * 0.0001
			})
		})
	}

	boundaryCandidates.sort((a, b) => a.score - b.score)
	const chosenBoundaryOpenings = new Set()
	const stride = Math.max(1, Math.floor(boundaryCandidates.length / entranceCount))
	for (let index = 0; index < boundaryCandidates.length && chosenBoundaryOpenings.size < entranceCount; index += stride) {
		const candidate = boundaryCandidates[index]
		chosenBoundaryOpenings.add(`${candidate.col}:${candidate.row}:${candidate.dir}`)
	}

	if (chosenBoundaryOpenings.size === 0 && boundaryCandidates[0]) {
		chosenBoundaryOpenings.add(`${boundaryCandidates[0].col}:${boundaryCandidates[0].row}:${boundaryCandidates[0].dir}`)
	}

	const primitives = []

	for (const cell of cellEntries) {
		directions.forEach((direction) => {
			const neighbor = cells.get(cellKey(cell.col + direction.dx, cell.row + direction.dz))
			const boundaryOpeningKey = `${cell.col}:${cell.row}:${direction.dir}`

			if (neighbor) {
				const isConnected = openEdges.has(edgeKey(cell.col, cell.row, neighbor.col, neighbor.row))
				if (isConnected) return
				if (direction.dir === 'west' || direction.dir === 'north') return
			} else if (chosenBoundaryOpenings.has(boundaryOpeningKey)) {
				const markerX = cell.x + direction.dx * (corridorWidth * 0.5 + wallThickness * 0.25)
				const markerZ = cell.z + direction.dz * (corridorWidth * 0.5 + wallThickness * 0.25)
				const markerWidth = direction.dx === 0 ? corridorWidth * 0.82 : wallThickness * 0.9
				const markerDepth = direction.dx === 0 ? wallThickness * 0.9 : corridorWidth * 0.82
				if (intersectsCampusRect(markerX, markerZ, markerWidth, markerDepth)) return

				primitives.push({
					kind: 'box',
					key: `marker-${cell.col}-${cell.row}-${direction.dir}`,
					position: [markerX, 0.05, markerZ],
					size: [markerWidth, 0.08, markerDepth],
					color: isNight ? '#a08a55' : '#d2b46d',
					material: 'ground',
					receiveShadow: true
				})
				return
			}

			const isVertical = direction.dir === 'east' || direction.dir === 'west'
			const wallX = cell.x + direction.dx * (corridorWidth * 0.5 + wallThickness * 0.5)
			const wallZ = cell.z + direction.dz * (corridorWidth * 0.5 + wallThickness * 0.5)
			const wallWidth = isVertical ? wallThickness : corridorWidth + wallThickness * 2
			const wallDepth = isVertical ? corridorWidth + wallThickness * 2 : wallThickness
			if (intersectsCampusRect(wallX, wallZ, wallWidth, wallDepth)) return

			primitives.push({
				kind: 'box',
				key: `wall-${cell.col}-${cell.row}-${direction.dir}`,
				position: [wallX, wallHeight / 2, wallZ],
				size: [wallWidth, wallHeight, wallDepth],
				color: isNight ? '#a79f92' : neighbor ? '#d7cfbf' : '#c9bea9',
				material: 'backdrop-block',
				castShadow: true,
				receiveShadow: true
			})
		})
	}

	return primitives
}

export default {
	createProviders({ pluginId, campusAssets }) {
		return campusAssets.map((asset) => ({
			id: `${pluginId}:labyrinth-walls`,
			sceneVariant: asset.defaults.backdrop?.sceneVariant || 'labyrinth',
			displayName: asset.name,
			description: asset.description,
			matchesAsset: (candidate) => candidate.id === asset.id || candidate.defaults.templateId === asset.defaults.templateId,
			createDefaultWorld: () => createDefaultLabyrinthWorld(),
			normalizeWorld: (raw) => ({
				...createDefaultLabyrinthWorld(),
				...(raw || {}),
				kind: 'labyrinth-walls'
			}),
			createScenePrimitives: ({ campusBounds, backdropRadius, isNight, world }) => createLabyrinthScenePrimitives(
				campusBounds,
				backdropRadius,
				{
					...createDefaultLabyrinthWorld(),
					...(world || {}),
					kind: 'labyrinth-walls'
				},
				isNight
			)
		}))
	}
}