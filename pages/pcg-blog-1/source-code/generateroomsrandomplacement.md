```cpp
std::vector<Polygon> DungeonGenerator::GenerateRoomsRandomPlacement(const Polygon& area, const size_t attempts, const int minWidth, const int maxWidth, const int minHeight, const int maxHeight)
{
	PCGDungeons::nextRoomId = 2;
	std::vector<Polygon> polygons;

	const float areaLeft = area.m_left, areaRight = area.m_right, areaTop = area.m_top, areaBottom = area.m_bottom;

	// Create grid
	const int width = static_cast<int>(area.m_width);
	const int xOffset = -static_cast<int>(area.m_left);
	const int yOffset = -static_cast<int>(area.m_bottom);
	Grid grid(width, static_cast<int>(area.m_height));

	auto position = glm::vec2(0.0f);
	for (size_t i = 0; i < attempts; ++i)
	{
		position.x = static_cast<float>(RandomInt(areaLeft, areaRight));
		position.y = static_cast<float>(RandomInt(areaBottom, areaTop));
		const Polygon polygon = CreateRandomRectangle(minWidth, maxWidth, minHeight, maxHeight, position);

		if (polygon.m_left < areaLeft || polygon.m_right > areaRight
			|| polygon.m_top > areaTop || polygon.m_bottom < areaBottom)
			continue;

		if (!OverlapRoomWithGrid(grid, polygon, xOffset, yOffset, width))
		{
			PlaceRoomInGrid(grid, polygon, xOffset, yOffset, width);
			polygons.push_back(polygon);
		}
	}

	return polygons;
}
```