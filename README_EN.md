# Operating System Memory Visual Simulator

**Language / 语言:** [English](./README_EN.md) | [简体中文](./README.md)

---

An interactive Vue 3-based visual simulator for operating system memory allocation algorithms, supporting dynamic demonstration and performance comparison of various memory allocation algorithms.

## 🎯 Core Features

### Feature 1: Dynamic Visual Simulation of Multiple Memory Allocation/Deallocation Algorithms
- **First Fit Algorithm**: Search for the first block large enough from the beginning of memory
- **Best Fit Algorithm**: Find the smallest block that is large enough to reduce memory waste
- **Worst Fit Algorithm**: Select the largest free block for allocation
- **Next Fit Algorithm**: Start searching for free blocks from the last allocation position

### Feature 2: Interactive User Operations and Parameter Configuration
- **Memory Configuration**: Customize total memory size (100KB - 2048KB)
- **Job Management**: Add, delete, and view job queues
- **Real-time Control**: Manual step-by-step execution, pause/resume, reset simulation
- **Dynamic Release**: Release allocated job memory at any time

## 🚀 Advanced Features

### Memory Fragmentation Analysis
- Real-time display of fragment count and fragmentation rate
- Maximum free block size monitoring
- Average fragment size statistics

### Algorithm Performance Comparison
- Visual performance charts
- Average allocation time statistics
- Successful allocation rate monitoring
- Memory utilization efficiency analysis

### Preset Test Scenarios
- **Small Task Intensive**: Test fragmentation with multiple small tasks
- **Large Task Impact**: Adaptability test with mixed large and small tasks
- **Incremental Sequence**: Test best/worst fit algorithm performance
- **Memory Stress Test**: Allocation testing near memory limits

### Automatic Demo Mode
- Adjustable playback speed (100ms - 2000ms)
- Automatic job queue execution
- Support for pause and stop controls

### History Recording and Playback
- Save memory state for each operation step
- Support forward/backward browsing of history
- One-click restore to any historical state

### Data Export Features
- **JSON Configuration Export**: Save current configuration and state
- **Image Export**: Export memory visualization images
- **Report Generation**: Generate detailed performance analysis reports
- **Configuration Import**: Load previously saved configurations

## 📦 Installation and Running

### Environment Requirements
- Node.js 16.0+
- npm or yarn

### Install Dependencies
```bash
npm install
```

### Run in Development Mode
```bash
npm run dev
```

### Build Production Version
```bash
npm run build
```

### Preview Production Version
```bash
npm run preview
```

## 🎮 Usage Guide

### Basic Operations

1. **System Configuration**
   - Set total memory size
   - Select allocation algorithm

2. **Add Jobs**
   - Enter job name (e.g., P1, P2...)
   - Set job size (KB)
   - Click "Add Job"

3. **Start Simulation**
   - Click "Start Simulation" to begin
   - Use "Next Step" for manual execution
   - Or use automatic mode for batch execution

4. **Observe Results**
   - View memory block visualization
   - Monitor statistical data
   - Read execution logs

### Advanced Operations

1. **Use Preset Scenarios**
   - Select test scenarios to start quickly
   - Observe performance differences between algorithms

2. **Automatic Demo**
   - Adjust playback speed
   - Start automatic playback mode
   - Real-time observation of allocation process

3. **History Playback**
   - Browse operation history
   - Compare different states
   - Analyze algorithm behavior

4. **Data Analysis**
   - View fragmentation statistics
   - Compare algorithm performance
   - Export analysis reports

## 🎨 Interface Description

### Control Panel
- **System Configuration**: Memory size and algorithm selection
- **Job Management**: Job queue and allocated jobs
- **Control Buttons**: Simulation control and state management

### Visualization Area
- **Memory Status**: Real-time statistical information
- **Memory Blocks**: Intuitive memory allocation diagram
- **Execution Log**: Detailed operation records

### Advanced Features Area
- **Fragmentation Analysis**: Memory usage efficiency
- **Performance Comparison**: Algorithm performance charts
- **Scenario Testing**: Preset test cases
- **Automatic Mode**: Batch execution control
- **History Records**: State playback functionality
- **Data Export**: Configuration and report export

## 🔧 Technical Implementation

### Core Technology Stack
- **Vue 3**: Composition API + TypeScript
- **Responsive Design**: Support for mobile and desktop
- **Canvas Drawing**: Performance chart visualization
- **File Processing**: Configuration import/export

### Algorithm Implementation
- **Memory Block Management**: Dynamic splitting and merging
- **Address Calculation**: Continuous memory address allocation
- **Fragment Defragmentation**: Automatic merging of adjacent free blocks
- **State Management**: Complete history recording system

### Visualization Features
- **Real-time Updates**: Reactive data binding
- **Color Coding**: Different jobs use different colors
- **Animation Effects**: Smooth state transitions
- **Interactive Feedback**: Mouse hover and click feedback

## 📚 Educational Value

### Learning Objectives
- Understand operating system memory management principles
- Master characteristics of different memory allocation algorithms
- Observe algorithm performance in different scenarios
- Analyze memory fragmentation problems and solutions

### Application Scenarios
- **Operating System Courses**: Memory management chapter demonstrations
- **Algorithm Learning**: Practical applications of greedy algorithms
- **Performance Analysis**: Effect comparison of different strategies
- **Experimental Teaching**: Interactive experiments and verification

## 🤝 Contribution Guidelines

Welcome to submit Issues and Pull Requests to improve this project!

### Development Suggestions
- Follow Vue 3 best practices
- Maintain TypeScript type safety
- Add appropriate comments and documentation
- Ensure responsive design compatibility

## 📄 License

MIT License

## 🙏 Acknowledgments

Thanks to all educators and developers who contribute to operating system education!

---

**Start exploring the mysteries of memory management!** 🚀
