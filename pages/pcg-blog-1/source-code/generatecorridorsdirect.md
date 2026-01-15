[Return to article](/pages/pcg-blog-1/blog.html#random-walk)

```cpp
std::vector<Corridor> DungeonGenerator::GenerateCorridorsDirect(const Polygon& area, const std::vector<Polygon>& rooms, const int allowOverlapPercentage, const int maxCorridorAttempts)
{
	if (rooms.empty())
	{
		bee::Log::Error("No rooms");
		return {};
	}
	const size_t roomCount = rooms.size();

	// Create and fill grid
	const int width = static_cast<int>(area.m_width);
	const int xOffset = -static_cast<int>(area.m_left);
	const int yOffset = -static_cast<int>(area.m_bottom);
	Grid grid(width, static_cast<int>(area.m_height));
	for (const Polygon& room : rooms)
		PlaceRoomInGrid(grid, room, xOffset, yOffset, width);

	std::vector<Polygon> roomsToConnect = rooms;
	std::vector<Polygon> connectedRooms;
	connectedRooms.reserve(roomCount);
	connectedRooms.push_back(rooms[0]);
	roomsToConnect.erase(roomsToConnect.begin());

	std::vector<Corridor> corridors;
	corridors.reserve(roomCount);

	int startX = 0, startY = 0;
	int endX = 0, endY = 0;
	int startLeft = 0, startRight = 0, startTop = 0, startBottom = 0;

	int corridorAttempts = 0;

	while (!roomsToConnect.empty())
	{
		const Polygon& goalPolygon = roomsToConnect[RandomInt(0, static_cast<int>(roomsToConnect.size()) - 1)];
		endX = RandomInt(goalPolygon.m_left + xOffset, goalPolygon.m_right + xOffset - 1);
		endY = RandomInt(goalPolygon.m_bottom + yOffset, goalPolygon.m_top + yOffset - 1);
		const Polygon& startPolygon = connectedRooms[RandomInt(0, static_cast<int>(connectedRooms.size()) - 1)];
		startLeft = static_cast<int>(startPolygon.m_left) + xOffset, startRight = static_cast<int>(startPolygon.m_right - 1) + xOffset;
		startTop = static_cast<int>(startPolygon.m_top) + yOffset - 1, startBottom = static_cast<int>(startPolygon.m_bottom) + yOffset;
		if (endX >= startLeft && endX <= startRight)
			startX = endX;
		else
			startX = RandomInt(startLeft, startRight);
		if (endY >= startBottom && endY <= startTop)
			startY = endY;
		else
			startY = RandomInt(startBottom, startTop);

		bool allowOverlap = RandomBoolPercent(allowOverlapPercentage);

		// Walk the corridor until a room is found
		Corridor corridor;
		corridor.m_vertices.reserve(abs(startX - endX) + abs(startY - endY));
		int foundRoomId = WalkCorridor(grid, startPolygon.m_roomId, startX, startY, endX, endY, width, allowOverlap, corridor);
		if (foundRoomId == Cell::errorCell.roomId || corridor.m_vertices.empty()) // No room found
		{
			corridorAttempts++;
			if (corridorAttempts >= maxCorridorAttempts) // Force allow overlap if attempts exceeds max
			{
				bee::Log::Error("Max corridor attempts exceeded!");
				// Clear corridor and try again but this time force allow overlap to true
				// Also walk from goal to start instead of other way around this time
				corridor.m_vertices.clear();
				foundRoomId = WalkCorridor(grid, goalPolygon.m_roomId, endX, endY, startX, startY, width, /*allowOverlap*/ true, corridor);
				if (foundRoomId == Cell::errorCell.roomId || corridor.m_vertices.empty()) // Still unsuccessful
				{
					bee::Log::Error("Search is still unsuccessful");
					continue;
				}
				else
				{
					size_t roomIndex = 0;
					for (; roomIndex < roomsToConnect.size(); ++roomIndex)
						if (roomsToConnect[roomIndex].m_roomId == goalPolygon.m_roomId)
							break;
					if (roomIndex != roomsToConnect.size())
						roomsToConnect.erase(roomsToConnect.begin() + roomIndex);
				}
			}
			else // If attempts does not exceed max, just retry
				continue;
		}
		corridorAttempts = 0; // After successful attempt, set attempts to 0
		CleanCorridor(corridor);
		for (auto& vertex : corridor.m_vertices)
		{
			vertex.x -= xOffset;
			vertex.y -= yOffset;
			vertex.x += 0.5f;
			vertex.y += 0.5f;
		}
		corridors.push_back(corridor);
		// Look for room with this foundRoom as m_roomChar
		size_t roomIndex = FindRoomByIDBinary(rooms, foundRoomId);
		if (roomIndex == rooms.size())
		{
			bee::Log::Error("Couldn't find room with roomId {}", foundRoomId);
			return {};
		}

		// Add room to connectedRooms
		connectedRooms.push_back(rooms[roomIndex]);

		// Delete room from roomsToConnect
		roomIndex = 0;
		for (roomIndex = 0; roomIndex < roomsToConnect.size(); ++roomIndex)
			if (roomsToConnect[roomIndex].m_roomId == foundRoomId)
				break;
		if (roomIndex != roomsToConnect.size())
			roomsToConnect.erase(roomsToConnect.begin() + roomIndex);
	}

	return corridors;
}

int DungeonGenerator::WalkCorridor(Grid& grid, const int ignore, const int startX, const int startY, const int endX, const int endY, const int gridWidth, const bool allowOverlap, Corridor& corridor)
{
	int x = startX, y = startY;
	int gridY = y * gridWidth;
	if (startX + gridY > static_cast<int>(grid.GetSize())) return Cell::errorCell.roomId;
	Cell* cell = grid.GetCellPointer(x + gridY);
	const int xDirection = (x < endX) ? 1 : -1;
	const int yDirection = (y < endY) ? 1 : -1;
	const int yStep = yDirection * gridWidth;
	corridor.m_vertices.push_back(glm::vec2(x, y));
	while (true)
	{
		if (x == endX && y == endY)
			return cell->roomId;
		if (x != endX && y != endY)
		{
			if (RandomBool())
			{
				x += xDirection;
				cell += xDirection;
			}
			else
			{
				y += yDirection;
				cell += yStep;
			}
		}
		else if (x == endX)
		{
			y += yDirection;
			cell += yStep;
		}
		else
		{
			x += xDirection;
			cell += xDirection;
		}

		switch (cell->type)
		{
		case TileType::Error:
			return Cell::errorCell.roomId;
		case TileType::Empty:
			corridor.m_vertices.push_back(glm::vec2(x, y));
			break;
		case TileType::Corridor:
			if (!allowOverlap) return Cell::errorCell.roomId;
			corridor.m_vertices.push_back(glm::vec2(x, y));
			break;
		default:
		{
			if (cell->roomId == ignore)
			{
				corridor.m_vertices.clear();
				corridor.m_vertices.push_back(glm::vec2(x, y));
				continue;
			}
			corridor.m_vertices.push_back(glm::vec2(x, y));
			for (size_t i = 1; i < corridor.m_vertices.size() - 1; ++i)
			{
				const auto& vertex = corridor.m_vertices[i];
				grid.SetCorridor(static_cast<int>(vertex.x), static_cast<int>(vertex.y));
			}
			return cell->roomId;
		}
		}
	}
}
```