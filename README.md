import { useState } from "react";
import { Card, CardContent } from "@/components/ui/card";
import { Button } from "@/components/ui/button";
import { Slider } from "@/components/ui/slider";
import { LineChart, Line, XAxis, YAxis, Tooltip, CartesianGrid, ResponsiveContainer, Legend } from "recharts";
import { format, isAfter, endOfDay } from "date-fns";

const getDaysInMonth = (month, year) => {
  return new Date(year, month + 1, 0).getDate();
};

export default function HabitTracker() {
  const [currentDate, setCurrentDate] = useState(new Date());
  const [progressData, setProgressData] = useState({});
  const [breakDays, setBreakDays] = useState({});
  const [taskData, setTaskData] = useState({});
  const [tasks, setTasks] = useState([]);
  const [newTask, setNewTask] = useState("");
  const [showProgressLine, setShowProgressLine] = useState(true);
  const [showTaskLine, setShowTaskLine] = useState(true);
  const [showTaskDelete, setShowTaskDelete] = useState(true);
  const [viewMode, setViewMode] = useState("monthly");

  const today = new Date();

  const handleSliderChange = (day, value) => {
    const key = format(currentDate, "yyyy-MM") + `-${day}`;
    setProgressData((prev) => ({ ...prev, [key]: value[0] }));
  };

  const toggleBreakDay = (day) => {
    const key = format(currentDate, "yyyy-MM") + `-${day}`;
    setBreakDays((prev) => ({ ...prev, [key]: !prev[key] }));
  };

  const handleAddTask = () => {
    if (newTask.trim()) {
      setTasks((prev) => [...prev, newTask.trim()]);
      setNewTask("");
    }
  };

  const handleTaskKeyDown = (e) => {
    if (e.key === "Enter") {
      handleAddTask();
    }
  };

  const handleTaskComplete = (index) => {
    setTasks((prev) => prev.filter((_, i) => i !== index));
  };

  const renderDays = () => {
    const month = currentDate.getMonth();
    const year = currentDate.getFullYear();
    const days = getDaysInMonth(month, year);
    const elements = [];
    for (let day = 1; day <= days; day++) {
      const dateObj = new Date(year, month, day);
      if (isAfter(dateObj, endOfDay(today))) continue;
      const key = format(currentDate, "yyyy-MM") + `-${day}`;
      const value = progressData[key] || 0;
      const isBreak = breakDays[key];

      let bg = isBreak ? "bg-blue-400" : "bg-gray-300";
      if (!isBreak) {
        if (value >= 80) bg = "bg-green-500";
        else if (value >= 30) bg = "bg-yellow-400";
      }

      elements.push(
        <div key={day} className="p-2 text-center cursor-pointer" onClick={() => toggleBreakDay(day)}>
          <div className={`w-10 h-10 rounded-full flex items-center justify-center text-white ${bg}`}>
            {day}
          </div>
          <Slider
            defaultValue={[value]}
            max={100}
            step={10}
            onValueChange={(val) => handleSliderChange(day, val)}
          />
        </div>
      );
    }
    return elements;
  };

  const getGraphData = () => {
    const month = format(currentDate, "yyyy-MM");
    const days = Object.entries(progressData)
      .filter(([key]) => key.startsWith(month))
      .map(([key, value]) => {
        return {
          date: key.split("-")[2],
          progress: value,
          task: 100
        };
      });
    return days;
  };

  const getSummary = () => {
    const month = format(currentDate, "yyyy-MM");
    const summary = {
      productive: [],
      medium: [],
      none: [],
      break: []
    };
    for (let day = 1; day <= getDaysInMonth(currentDate.getMonth(), currentDate.getFullYear()); day++) {
      const dateObj = new Date(currentDate.getFullYear(), currentDate.getMonth(), day);
      if (isAfter(dateObj, endOfDay(today))) continue;
      const key = `${month}-${day}`;
      if (breakDays[key]) {
        summary.break.push(day);
        continue;
      }
      const value = progressData[key] || 0;
      if (value >= 80) summary.productive.push(day);
      else if (value >= 30) summary.medium.push(day);
      else summary.none.push(day);
    }
    return summary;
  };

  const changeMonth = (offset) => {
    const newDate = new Date(currentDate);
    newDate.setMonth(currentDate.getMonth() + offset);
    setCurrentDate(newDate);
  };

  const summary = getSummary();

  return (
    <div className="p-6 space-y-6 relative">
      <div className="flex justify-between items-center">
        <Button onClick={() => changeMonth(-1)}>Previous</Button>
        <h2 className="text-xl font-bold">
          {format(currentDate, "MMMM yyyy")}
        </h2>
        <Button onClick={() => changeMonth(1)}>Next</Button>
      </div>

      <div className="flex justify-end gap-4">
        <label className="flex items-center gap-2">
          <input type="radio" value="monthly" checked={viewMode === "monthly"} onChange={() => setViewMode("monthly")} />
          Monthly View
        </label>
        <label className="flex items-center gap-2">
          <input type="radio" value="weekly" checked={viewMode === "weekly"} onChange={() => setViewMode("weekly")} />
          Weekly View
        </label>
      </div>

      <Card>
        <CardContent className="grid grid-cols-7 gap-4 pt-4">
          {renderDays()}
        </CardContent>
      </Card>

      <Card>
        <CardContent className="pt-4 space-y-4">
          <h3 className="text-lg font-semibold">Productivity Summary</h3>
          <div>
            <p><strong>Best Days:</strong> {summary.productive.join(", ") || "None"}</p>
            <p><strong>Medium Days:</strong> {summary.medium.join(", ") || "None"}</p>
            <p><strong>No Progress:</strong> {summary.none.join(", ") || "None"}</p>
            <p><strong>Break Days:</strong> {summary.break.join(", ") || "None"}</p>
          </div>
        </CardContent>
      </Card>

      <Card>
        <CardContent className="pt-4 space-y-2">
          <h3 className="text-lg font-semibold mb-2">Consistency Graph</h3>
          <div className="flex gap-4 mb-2">
            <label className="flex items-center gap-2">
              <input type="checkbox" checked={showProgressLine} onChange={() => setShowProgressLine(!showProgressLine)} />
              Show Productivity
            </label>
            <label className="flex items-center gap-2">
              <input type="checkbox" checked={showTaskLine} onChange={() => setShowTaskLine(!showTaskLine)} />
              Show Task
            </label>
          </div>
          <ResponsiveContainer width="100%" height={300}>
            <LineChart data={getGraphData()}>
              <CartesianGrid strokeDasharray="3 3" />
              <XAxis dataKey="date" />
              <YAxis domain={[0, 100]} />
              <Tooltip />
              <Legend />
              {showProgressLine && <Line type="monotone" dataKey="progress" stroke="#10b981" strokeWidth={2} dot={{ r: 4 }} activeDot={{ r: 6 }} name="Productivity" />}
              {showTaskLine && <Line type="monotone" dataKey="task" stroke="#3b82f6" strokeWidth={2} dot={{ r: 4 }} activeDot={{ r: 6 }} name="Task" />}
            </LineChart>
          </ResponsiveContainer>
        </CardContent>
      </Card>

      <div className="absolute bottom-4 right-4 w-64 p-4 bg-white border rounded-xl shadow-xl space-y-2">
        <h4 className="font-semibold text-lg">Daily Tasks</h4>
        <input
          type="text"
          value={newTask}
          onChange={(e) => setNewTask(e.target.value)}
          onKeyDown={handleTaskKeyDown}
          placeholder="Enter new task"
          className="w-full p-2 border rounded"
        />
        <Button onClick={handleAddTask} className="w-full">Add Task</Button>
        <label className="flex items-center gap-2 text-sm">
          <input type="checkbox" checked={showTaskDelete} onChange={() => setShowTaskDelete(!showTaskDelete)} />
          Enable Delete on Complete
        </label>
        <ul className="list-none max-h-40 overflow-y-auto space-y-1">
          {tasks.map((task, index) => (
            <li key={index} className="flex items-center gap-2">
              <input type="checkbox" onChange={() => showTaskDelete && handleTaskComplete(index)} />
              <span>{task}</span>
            </li>
          ))}
        </ul>
      </div>
    </div>
  );
}
