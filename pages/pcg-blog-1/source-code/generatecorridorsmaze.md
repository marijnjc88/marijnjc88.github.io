```cpp
std::vector<Corridor> DungeonGenerator::GenerateCorridorsMaze(const Polygon& area, const std::vector<Polygon>& rooms)
{
	if (rooms.empty())
	{
		bee::Log::Error("No rooms");
		return {};
	}

	const int width = static_cast<int>(area.m_width);
	const int height = static_cast<int>(area.m_height);
	const int xOffset = -static_cast<int>(area.m_left);
	const int yOffset = -static_cast<int>(area.m_bottom);
	Grid grid(width, height);
	for (const Polygon& room : rooms)
		PlaceRoomInGrid(grid, room, xOffset, yOffset, width);

	std::vector<Corridor> corridors;
	corridors.reserve(static_cast<size_t>(area.m_area * 0.125f) + rooms.size());
	Corridor pathStack;

	int gridY = 0;
	// Allocate once then reuse
	std::vector<glm::ivec2> validDirections;
	validDirections.reserve(3);
	// Maze generation algorithm here
	for (int masterY = 0; masterY < height; ++masterY)
	{
		gridY = masterY * width;
		for (int masterX = 0; masterX < width; ++masterX)
		{
			if (grid.GetCell(masterX + gridY).type != TileType::Empty)
				continue;

			pathStack.m_vertices.clear();
			// Maze generation algorithm
			Corridor corridor;
			int x = masterX, y = masterY;
			grid.SetCorridor(x, y);
			corridor.m_vertices.push_back(glm::vec2(x - xOffset + 0.5f, y - yOffset + 0.5f));
			pathStack.m_vertices.push_back(glm::vec2(x, y));
			while (true)
			{
				GetValidDirections(grid, x, y, validDirections);
				// No valid directions
				if (validDirections.empty())
				{
					if (corridor.m_vertices.size() >= 2)
					{
						CleanCorridor(corridor);
						corridors.push_back(corridor);
					}
					while (pathStack.m_vertices.size() > 0)
					{
						const auto& point = pathStack.m_vertices.back();
						GetValidDirections(grid, static_cast<int>(point.x), static_cast<int>(point.y), validDirections);
						if (!validDirections.empty())
						{
							x = static_cast<int>(point.x), y = static_cast<int>(point.y);
							corridor.m_vertices.clear();
							corridor.m_vertices.push_back(glm::vec2(x - xOffset + 0.5f, y - yOffset + 0.5f));
							break;
						}
						else
							pathStack.m_vertices.pop_back();
					}
					if (pathStack.m_vertices.empty()) // Once entire path stack is empty, break the loop (maze is finished)
						break;
				}
				// At least one valid direction
				else
				{
					const auto& direction = (validDirections.size() > 1) ? validDirections[static_cast<size_t>(RandomInt(0, static_cast<int>(validDirections.size()) - 1))] : validDirections[0];
					x += static_cast<int>(direction.x), y += static_cast<int>(direction.y);
					grid.SetCorridor(x, y);
					corridor.m_vertices.push_back(glm::vec2(x - xOffset + 0.5f, y - yOffset + 0.5f));
					pathStack.m_vertices.push_back(glm::vec2(x, y));
				}
			}
		}
	}

	// Connect rooms to maze
	int roomTop = 0, roomRight = 0, roomBottom = 0, roomLeft = 0, rand = 0;
	int areaTop = static_cast<int>(area.m_top) - 1;
	int areaRight = static_cast<int>(area.m_right) - 1;
	int areaBottom = static_cast<int>(area.m_bottom);
	int areaLeft = static_cast<int>(area.m_left);
	for (const auto& room : rooms)
	{
		roomTop = static_cast<int>(room.m_top) - 1;
		roomRight = static_cast<int>(room.m_right) - 1;
		roomBottom = static_cast<int>(room.m_bottom);
		roomLeft = static_cast<int>(room.m_left);
		validDirections.clear();
		if (roomTop < areaTop)
			validDirections.push_back(Direction::Up);
		if (roomRight < areaRight)
			validDirections.push_back(Direction::Right);
		if (roomBottom > areaBottom)
			validDirections.push_back(Direction::Down);
		if (roomLeft > areaLeft)
			validDirections.push_back(Direction::Left);
		glm::ivec2 direction = validDirections[RandomInt(0, static_cast<int>(validDirections.size()) - 1)];
		corridors.emplace_back();
		auto& corridor = corridors.back();
		corridor.m_vertices.reserve(2);
		if (direction == Direction::Up)
		{
			rand = RandomInt(roomLeft, roomRight);
			corridor.m_vertices.push_back(glm::vec2(rand + 0.5f, roomTop + 0.5f));
			corridor.m_vertices.push_back(glm::vec2(rand + 0.5f, roomTop + 1 + 0.5f));
		}
		else if (direction == Direction::Right)
		{
			rand = RandomInt(roomBottom, roomTop);
			corridor.m_vertices.push_back(glm::vec2(roomRight + 0.5f, rand + 0.5f));
			corridor.m_vertices.push_back(glm::vec2(roomRight + 0.5f + 1, rand + 0.5f));
		}
		else if (direction == Direction::Down)
		{
			rand = RandomInt(roomLeft, roomRight);
			corridor.m_vertices.push_back(glm::vec2(rand + 0.5f, roomBottom + 0.5f));
			corridor.m_vertices.push_back(glm::vec2(rand + 0.5f, roomBottom - 1 + 0.5f));
		}
		else
		{
			rand = RandomInt(roomBottom, roomTop);
			corridor.m_vertices.push_back(glm::vec2(roomLeft + 0.5f, rand + 0.5f));
			corridor.m_vertices.push_back(glm::vec2(roomLeft + 0.5f - 1, rand + 0.5f));
		}
	}

	return corridors;
}

void DungeonGenerator::GetValidDirections(Grid& grid, int x, int y, std::vector<glm::ivec2>& outputList)
{
	outputList.clear();
	for (auto direction : Direction::AllDirections)
	{
		int newX = x + direction.x, newY = y + direction.y;
		if (!grid.IsValidX(newX) || !grid.IsValidY(newY))
			continue;
		if (grid.GetCell(newX, newY).type == TileType::Empty)
			outputList.push_back(direction);
	}
}
```