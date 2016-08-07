---
layout: default
title: AStarDungeon (C++)
category: misc
---
<div class="content-inner">
    <div class="container">
        <h1>AStarDungeon</h1>

        <h3>Programming language:</h3>
        <p>
        	C++
        </p>

        <h3>Github:</h3>
        <p>
        <a href="https://github.com/gpanic/AStarDungeon">https://github.com/gpanic/AStarDungeon</a>
        </p>

        <h3>Description</h3>
        <p>
          A simple A* implementation used in a programming challenge. It works on a map in form of a grid,
          with marked start (S), goal (G) and wall (W) tiles. It finds the shortest path and prints an updated map.
      	</p>
      	<p>
          The class AStar is a generic implementation of the A* search algorithm. The class AStarDungeon contains the logic to process the map, use AStar to find the path and print the updated map.
      	</p>

      	<h3>Input and output</h3>
{% highlight cpp %}
WW..W...............            WW..W...............
WWW.W....G.WW.......            WWW.W..**G.WW.......
WWW.W...WWWWW.WWW...            WWW.W..*WWWWW.WWW...
...............WW.W.            .......****....WW.W.
..WWW...WW..WWWWW.WW            ..WWW...WW*.WWWWW.WW
....W...W...WWWWW..W            ....W...W.*.WWWWW..W
WW..W..WWW..WWWWW..W            WW..W..WWW*.WWWWW..W
WW..W..WWW..W.......            WW..W..WWW*.W.......
....W...............            ....W.....****......
WWWWWWWWWWWWW..WWW.W            WWWWWWWWWWWWW*.WWW.W
........W..........W            ........W.****.....W
WWWW..WWWW..WW..WWWW            WWWW..WWWW*.WW..WWWW
.............W..W...            ......*****..W..W...
WW.WWW..WWW..W..W.WW            WW.WWW*.WWW..W..W.WW
WW.W................            WW.W..*.............
WW.....WWWWW..WWWWWW            WW....*WWWWW..WWWWWW
W.......SW......WWW.            W.....**SW......WWW.
WW.......WW.......W.            WW.......WW.......W.
W........WW...WWWWW.            W........WW...WWWWW.
..................W.            ..................W.
                                Length: 32
{% endhighlight %}

      	<h3>AStar.h</h3>
{% highlight cpp %}
#pragma once
#include <queue>
#include <deque>
#include <unordered_set>
#include <unordered_map>

class AStar
{
public:
    class Node
    {
    public:
        int x, y;
        float f; // estimated cost
        float g; // actual cost to get here

        Node();
        Node(int x, int y);

        bool operator==(const Node &node) const;
        bool operator!=(const Node &node) const;

        // hashing functor
        struct Hash
        {
            // use coordinates for equality, a simple noncommutative hash function
            std::size_t operator()(const Node &node) const
            {
                return 3 * node.x + node.y;
            }
        };

        // comparison functor
        struct GreaterByCost
        {
            bool operator()(const Node &lhs, const Node &rhs) const
            {
                return lhs.f > rhs.f;
            }
        };
    };

protected:
    std::vector<Node> FindPath(Node start, Node goal);
    virtual std::vector<Node> GetSuccessors(const Node &node) const = 0;
    virtual float HeuristicFunction(const Node &node1, const Node &node) const = 0;

private:
    std::deque<Node> openList; // nodes to visit
    std::unordered_set<Node, Node::Hash> closedList; // visited nodes
    std::unordered_map<Node, Node, Node::Hash> cameFrom; // backwards connections

    std::vector<Node> ReconstructPath(Node currentNode);
};
{% endhighlight %}

		<h3>AStar.cpp</h3>
{% highlight cpp %}
#include "AStar.h"

AStar::Node::Node() : x(-1), y(-1) {}
AStar::Node::Node(int x, int y) : x(x), y(y) {}

bool AStar::Node::operator==(const Node &node) const
{
    return x == node.x && y == node.y;
}

bool AStar::Node::operator!=(const Node &node) const
{
    return !(*this == node);
}

// assumes monotonic heuristic, uses functions push_heap and pop_heap
// to get the node with the smallest estimate
std::vector<AStar::Node> AStar::FindPath(Node start, Node goal)
{
    start.f = start.g = 0;
    openList.push_back(start);

    Node currentNode;

    while (!openList.empty())
    {
        // get the node with the smallest estimate
        std::pop_heap(openList.begin(), openList.end(), Node::GreaterByCost());
        currentNode = openList.back();
        openList.pop_back();

        if (currentNode == goal) break;

        // node was visited
        closedList.insert(currentNode);

        std::vector<Node> successors = GetSuccessors(currentNode);
        for (Node successor : successors)
        {
            // skip already visited nodes
            if (closedList.find(successor) != closedList.end()) continue;

            // remember connection to successor
            cameFrom[successor] = currentNode;

            // update sucessor estimate and cost
            successor.g = currentNode.g + 1; // always move by one tile
            successor.f = successor.g + HeuristicFunction(successor, goal);

            // add successor to the open list if wasn't added in a previous iteration
            if (std::find(openList.begin(), openList.end(), successor) == openList.end())
            {
                openList.push_back(successor);
                std::push_heap(openList.begin(), openList.end(), Node::GreaterByCost());
            }
        }
    }

    return ReconstructPath(currentNode);
}

std::vector<AStar::Node> AStar::ReconstructPath(Node currentNode)
{
    std::vector<Node> path;
    path.push_back(currentNode);

    // go through the list of backwards connections until the starting node
    while (cameFrom.find(currentNode) != cameFrom.end())
    {
        currentNode = cameFrom[currentNode];
        path.push_back(currentNode);
    }

    return path;
}
{% endhighlight %}

		<h3>AStarDungeon.h</h3>
{% highlight cpp %}
#pragma once
#include "AStar.h"
#include <fstream>
#include <string>
#include <iostream>
#include <vector>
#include <tuple>

class AStarDungeon : AStar
{
public:
    bool LoadMap(const std::string mapFile);
    void MarkPathToGoal();
    void PrintMap() const;
    int GetPathLength();

private:
    enum TileType { typeNormal = '.', typeWall = 'W', typeGoal = 'G', typeStart = 'S', typePath = '*', typeInvalid = 'I' };

    std::vector<std::vector<char>> map;
    Node startNode;
    Node goalNode;
    bool mapLoaded = false;
    std::vector<Node> path;

    
    bool IsCharValid(const char &c) const;
    TileType GetTileType(const Node &node) const;
    bool CanMoveTo(const Node &node) const;

    std::vector<Node> GetSuccessors(const Node &node) const override;
    float HeuristicFunction(const Node &node1, const Node &node2) const override;
};
{% endhighlight %}

		<h3>AStarDungeon.cpp</h3>
{% highlight cpp %}
#include "AStarDungeon.h"

bool AStarDungeon::LoadMap(const std::string mapFile)
{
    std::ifstream file(mapFile);
    if (!file) return false;

    std::string line;
    bool foundStart = false;
    bool foundGoal = false;
    while (file >> line)
    {
        std::vector<char> mapRow; // each line represents a map row
        for (char c : line)
        {
            if (!IsCharValid(c)) return false;

            if (c == TileType::typeStart)
            {
                if (foundStart) // allow only one start
                {
                    return false;
                }
                else 
                {
                    startNode.x = mapRow.size();
                    startNode.y = map.size();
                    foundStart = true;
                }
            }
            else if (c == TileType::typeGoal)
            {
                if (foundGoal) // allow only one goal
                {
                    return false;
                }
                else
                {
                    goalNode.x = mapRow.size();
                    goalNode.y = map.size();
                    foundGoal = true;
                }
            }

            mapRow.push_back(c);
        }
        map.push_back(mapRow);
    }

    mapLoaded = foundStart && foundGoal;
    return mapLoaded;
}

void AStarDungeon::MarkPathToGoal()
{
    if (!mapLoaded) return;
    path = FindPath(startNode, goalNode);
    for (Node n : path)
    {
        char &tile = map.at(n.y).at(n.x);
        if (tile != TileType::typeGoal && tile != TileType::typeStart)
        {
            tile = TileType::typePath;
        }
    }
}

void AStarDungeon::PrintMap() const
{
    for (auto row = map.begin(); row != map.end(); ++row)
    {
        for (auto tile = row->begin(); tile != row->end(); ++tile)
        {
            std::cout << *tile;
        }
        std::cout << std::endl;
    }
}

int AStarDungeon::GetPathLength()
{
    if (!mapLoaded) return -1;
    return path.size() - 1; // we exclude the starting node
}

bool AStarDungeon::IsCharValid(const char &c) const
{
    return c == TileType::typeNormal ||
        c == TileType::typeWall ||
        c == TileType::typeGoal ||
        c == TileType::typeStart;
}

AStarDungeon::TileType AStarDungeon::GetTileType(const Node &node) const
{
    try
    {
        return static_cast<TileType>(map.at(node.y).at(node.x));
    }
    catch (std::out_of_range)
    {
        return TileType::typeInvalid;
    }
}

bool AStarDungeon::CanMoveTo(const Node &node) const
{
    TileType type = GetTileType(node);
    return type != TileType::typeWall && type != TileType::typeInvalid;
}

std::vector<AStarDungeon::Node> AStarDungeon::GetSuccessors(const AStarDungeon::Node &node) const
{
    Node up = node;
    ++up.y;
    Node right = node;
    ++right.x;
    Node down = node;
    --down.y;
    Node left = node;
    --left.x;

    std::vector<Node> successors = { up, right, down, left };

    // get rid of untraversable tiles
    auto canMoveToLambda = [this](Node &n) { return !CanMoveTo(n); };
    successors.erase(std::remove_if(successors.begin(), successors.end(), canMoveToLambda), successors.end());

    return successors;
}

// we use manhattan distance because it's monotonic, fast and simple
float AStarDungeon::HeuristicFunction(const Node &node1, const Node &node2) const
{
    return std::abs(node1.x - node2.x) + std::abs(node1.y - node2.y);
}
{% endhighlight %}

		<h3>Main.cpp</h3>
{% highlight cpp %}
#include "AStarDungeon.h"
#include "AStar.h"

// execute: astardungeon.exe map.txt
int main(int argc, const char* argv[])
{
    if (argc < 2)
    {
        std::cout << "Pass the path to a valid map as the first argument." << std::endl;
        return 1;
    }

    AStarDungeon pathFinder;
    if (!pathFinder.LoadMap(argv[1]))
    {
        std::cout << "The provided map was invalid." << std::endl;
    }
    pathFinder.MarkPathToGoal();
    pathFinder.PrintMap();

    std::cout << "Length: " << pathFinder.GetPathLength();

    std::cin.get();
    return 0;
}
{% endhighlight %}
    </div>
</div>