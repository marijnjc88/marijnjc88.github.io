```cpp
std::vector<Polygon> DungeonGenerator::GenerateRoomsBinary(const Polygon& area, const int maxRoomCount, const int minWidth, const int minHeight, const int trimSize, const BinaryDirectionBias directionBias)
{
	PCGDungeons::nextRoomId = 2;
	if (maxRoomCount <= 1) return { area };
	if (area.m_area <= 4.0f)
	{
		bee::Log::Error("Invalid area");
		return { area };
	}
	if (minWidth <= 0 || minHeight <= 0)
	{
		bee::Log::Error("Invalid minimum parameters");
		return { area };
	}
	if (trimSize <= 0)
	{
		bee::Log::Error("Invalid trim size");
		return { area };
	}

	std::vector<Polygon> polygons;
	polygons.reserve(maxRoomCount);

	std::priority_queue<Polygon, std::vector<Polygon>, ComparePolygonsByArea> polygonQueue;
	polygonQueue.push(area);

	const int doubleTrimSize = trimSize + trimSize;

	const float actualMinWidth = static_cast<float>(minWidth + doubleTrimSize);
	const float actualMinHeight = static_cast<float>(minHeight + doubleTrimSize);

	const float doubleMinWidth = static_cast<float>(actualMinWidth * 2);
	const float doubleMinHeight = static_cast<float>(actualMinHeight * 2);
	const float invMinWidth = 1.0f / static_cast<float>(actualMinWidth);
	const float invMinHeight = 1.0f / static_cast<float>(actualMinHeight);

	int roomCount = 1;
	bool prioritizeWidth = false;
	while (roomCount < maxRoomCount && !polygonQueue.empty())
	{
		const Polygon largest = polygonQueue.top();
		polygonQueue.pop();
		switch (directionBias)
		{
		case BinaryDirectionBias::LargestDirection:
		{
			const float widthWeight = largest.m_width * invMinWidth;
			const float heightWeight = largest.m_height * invMinHeight;
			if (widthWeight == heightWeight) prioritizeWidth = RandomBool();
			else prioritizeWidth = (widthWeight >= heightWeight);
			break;
		}

		case BinaryDirectionBias::RandomDirection:
		{
			prioritizeWidth = RandomBool();
			break;
		}
		}
		if (prioritizeWidth)
		{
			if (largest.m_width >= doubleMinWidth)
			{
				const auto split = HorizontalSplit(largest, actualMinWidth);
				polygonQueue.push(split.first);
				polygonQueue.push(split.second);
				roomCount++;
			}
			else if (largest.m_height >= doubleMinHeight)
			{
				const auto split = VerticalSplit(largest, actualMinHeight);
				polygonQueue.push(split.first);
				polygonQueue.push(split.second);
				roomCount++;
			}
			else // Polygon cannot be split any further in either direction
				polygons.push_back(largest);

		}
		else // prioritize height
		{
			if (largest.m_height >= doubleMinHeight)
			{
				const auto split = VerticalSplit(largest, actualMinHeight);
				polygonQueue.push(split.first);
				polygonQueue.push(split.second);
				roomCount++;
			}
			else if (largest.m_width >= doubleMinWidth)
			{
				const auto split = HorizontalSplit(largest, actualMinWidth);
				polygonQueue.push(split.first);
				polygonQueue.push(split.second);
				roomCount++;
			}
			else // Polygon cannot be split any further in either direction
				polygons.push_back(largest);
		}
	}

	while (!polygonQueue.empty())
	{
		polygons.push_back(polygonQueue.top());
		polygonQueue.pop();
	}

	const float fTrimSize = static_cast<float>(trimSize);
	for (auto& polygon : polygons)
		polygon.Trim(fTrimSize);

	// Sort polygons by id
	std::sort(polygons.begin(), polygons.end(), [](const Polygon& a, const Polygon& b) {return a.m_roomId < b.m_roomId; });

	return polygons;
}

std::pair<Polygon, Polygon> DungeonGenerator::HorizontalSplit(const Polygon& polygon, const float minWidth)
{
	const float polygonWidth = polygon.m_width;
	const float polygonHeight = polygon.m_height;
	const float polygonLeft = polygon.m_left;
	const float polygonBottom = polygon.m_bottom;

	const float split = static_cast<float>(RandomInt(minWidth, polygonWidth - minWidth));

	return { 
		Polygon(0.0f, split, polygonHeight, 0.0f, { polygonLeft, polygonBottom }),
		Polygon(0.0f, polygonWidth - split, polygonHeight, 0.0f, { polygonLeft + split, polygonBottom })
	};
}

std::pair<Polygon, Polygon> DungeonGenerator::VerticalSplit(const Polygon& polygon, const float minHeight)
{
	const float polygonWidth = polygon.m_width;
	const float polygonHeight = polygon.m_height;
	const float polygonLeft = polygon.m_left;
	const float polygonBottom = polygon.m_bottom;

	const float split = static_cast<float>(RandomInt(minHeight, polygonHeight - minHeight));

	return {
		Polygon(0.0f, polygonWidth, polygonHeight - split, 0.0f, { polygonLeft, polygonBottom + split }),
		Polygon(0.0f, polygonWidth, split, 0.0f, { polygonLeft, polygonBottom })
	};
}
```