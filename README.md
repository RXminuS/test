Peer review
===================
Lab 2 by Eskil Andersson
-------------
reviewed by Rik Nauta & Marcus Svenssons

----------
##Overview
The code presents a working solution to all the required functionality as well as all the optional improvements. The Monitor correctly does most of the 'heavy lifting'[^pun_footnote] and we haven't been able to find any race conditions, starvations, or deadlocks. Well done!

----------

##The Good
- All variables are well protected and synchronised methods are correctly utilised  
- Because of the level of abstraction chosen it's very easy to adjust the amount of floors or elevator specifications
- All optional improvements implemented
- This implementation, in contrast to ours, correctly places the <code>try/catch</code> block within the <code>while(condition)</code>. Thus an interrupt can never cause an incorrect continuation of the code-path.
- This implementation, again in contract to ours, correctly checks all conditions whithing the same <code>while</code> loop thus correctly preventing continuation of the code-path unless **all conditions are simultaniously met**.

----------

##The Bad

- The Monitor object does not provide it's own <code>main</code> routine and instead modifies the existing one in the LiftView object. Page 122 of the lab specification states *"You are not supposed to modify this main method. You should write your own main in your own class"*.
- In the current implementation there is erroneous behaviour because the elevator never takes into consideration people waiting on the current floor when calling <code>optimizeRoute</code>. 
>**Example:**
>When the elevator travelled from floor 2 to 1 with one person waiting on floor 1 to go to floor 0; the elevator decided to swap directions instead of continuing it's path down and move to floor 2 instead (where people were waiting also).

- There are <code>sleep()</code> blocks in the <code>floorUpdate()</code> method which seem to serve no purpose.

----------

##The Ugly
- The naming of functions and variables is very confusing and inconsistent. For example
	- From the name it is unclear what <code>okGo()</code>, <code>apply()</code> and <code>floorUpdate()</code> **actually** do.
- Even though in the current implementation it does not present a problem or concurrency errors, it could be a good idea to have the Monitor in control of all the rendering and avoid calling <code>drawLift()</code> in the <code>Lift</code>-thread, and thus requiring sharing the <code>LiftView</code> object between threads. 
- There is a lot of redundant data and duplicate code increasing the chances of forgetting to update related information or making refactoring unnecessarily hard.
	- The <code>doorsClosed</code> variable could easily be extracted from a difference in the current and target floors.
	- The mixing and separate storage of <code>topFloor</code> and <code>nrOfFloors</code> which relate to each other with a constant.
	- A person passes it's direction of travel when calling <code>apply( </code> which could easily be extracted from the difference between <code>start</code> and <code>destination</code>.
- A simple enumeration instead of a boolean value for movement direction would increase readability and decrease chances for erroneous interpretation.
- When opting for concatenated <code>if</code>-clauses (in contrast to our implementation where we split them into several clauses) care must be taken to split the clauses over several lines to improve readability.


----------

#Code
```
package lift;

public class Person extends Thread{
	
	private int destination, currentFloor, nrOfFloors;
	private Monitor m;
	
	public Person(Monitor m, int nrOfFloors) {
		this.nrOfFloors = nrOfFloors;
		this.m = m;
	}
	
	/*
	 Method getDirection() returns true if person is going up and false if person is going down.
	*/
	private boolean getDirection() {
		return destination - currentFloor > 0;
	}
	
	public void run() {
		while(true) {
			int delay = 1000*((int)(Math.random()*46.0));
			
			try {
				Thread.sleep(delay);
			} catch(InterruptedException e) {}
		
			currentFloor = (int)(Math.random()*nrOfFloors);
		
			do {
				destination = (int)(Math.random()*nrOfFloors);
			} while(destination == currentFloor);
		
			m.apply(currentFloor, destination, getDirection());
		}
	}
}

public class Elevator extends Thread {
	
	private LiftView lv;
	private Monitor m;
	private int nrOfFloors, currentFloor, nextFloor;
	private boolean goingUp;
	
	public Elevator(LiftView lv, Monitor m, int nrOfFloors) {
		this.lv = lv;
		this.m = m;
		this.nrOfFloors = nrOfFloors;
		currentFloor = 0;
		nextFloor = 1;
		goingUp = true;
	}
	
	private void setNextFloor() {
		if(goingUp) {
			nextFloor = currentFloor + 1;
		} else {
			nextFloor = currentFloor - 1;
		}
	}
	
	private void changeState() {
		if(goingUp) {
			goingUp = false;
		} else {
			goingUp = true;
		}
	}
	
	public void run() {
		lv.drawLift(currentFloor, 0);
		while(true) {
			if(m.floorUpdate(currentFloor, goingUp)) {
				changeState();
			}
			setNextFloor();
			lv.moveLift(currentFloor, nextFloor);
			currentFloor = nextFloor;
			
			if(currentFloor == nrOfFloors - 1 || currentFloor == 0) {
				changeState();
			}
			
			
		}
	}

}

public class Monitor {
	
	private int currentFloor, topFloor, maxLoad, loadInElevator, totalLoad;
	private int[] loadOnFloor, goingUpOnFloor, goingDownOnFloor, getsOffAt;
	private boolean doorsClosed, elevatorDirection;
	private LiftView lv;
	
	public Monitor(LiftView lv, int nrOfFloors, int maxLoad) {
		this.lv = lv;
		currentFloor = 0;
		topFloor = nrOfFloors - 1;
		loadOnFloor = new int[nrOfFloors]; // loadOnFloor[i] = goingUpOnFloor[i] + goingDownOnFloor[i].
		goingUpOnFloor = new int[nrOfFloors]; // Nr of persons on floor [i] who wants to go up.
		goingDownOnFloor = new int[nrOfFloors]; // Nr of person on floor [i] who wants to go down.
		getsOffAt = new int[nrOfFloors]; // Nr of persons who want to get off at the floor specified by index.
		loadInElevator = 0;
		totalLoad = 0;
		this.maxLoad = maxLoad;
		doorsClosed = true;
	}
	
	/*
	 * A Person-thread applies to the elevator and completes the travel, all optional implementations fulfilled.
	 */
	public synchronized void apply(int start, int destination, boolean direction) {
		if(direction) {
			goingUpOnFloor[start]++;
		} else {
			goingDownOnFloor[start]++;
		}
		loadOnFloor[start]++;
		totalLoad++;
		notifyAll();
		lv.drawLevel(start, loadOnFloor[start]);
		
		while(dontEnter(start, direction)) { 
			try {
				wait();
			} catch(InterruptedException e) {} 
		}
		
		getOn(start, destination);
		notifyAll();
		
		while(destination != currentFloor) {
			try {
				wait();
			} catch(InterruptedException e) {}
		} 
		
		getOff(destination);
		notifyAll();
	}
	
	/*
	 * The Elevator-thread updates all Person-threads when a new floor is reached.
	 */
	public synchronized boolean floorUpdate(int floorLevel, boolean elevatorDirection) {
		currentFloor = floorLevel;
		this.elevatorDirection = elevatorDirection;
		doorsClosed = false;
		
		try {
			Thread.sleep(100);
		} catch(InterruptedException e) {}
		
		notifyAll();
		
		while(!okGo()) {
			try {
				wait();
			} catch(InterruptedException e) {}
		}
		doorsClosed = true;
		try {
			Thread.sleep(300);
		} catch(InterruptedException e) {}
		return optimizeRoute(elevatorDirection);
	}
	
	/*
	 * The Person-thread gets on the elevator.
	 */
	private void getOn(int floor, int destination) { 
		if(elevatorDirection) {
			goingUpOnFloor[floor]--;
		} else {
			goingDownOnFloor[floor]--;
		}
		loadOnFloor[floor]--;
		loadInElevator++;
		getsOffAt[destination]++;
		lv.drawLevel(floor, loadOnFloor[floor]);
		lv.drawLift(floor, loadInElevator);
	}
	
	/*
	 * The Person-thread gets off the elevator.
	 */
	private void getOff(int floor) {
		loadInElevator--;
		lv.drawLift(floor, loadInElevator);
		getsOffAt[floor]--;
		totalLoad--;
	}
	
	private boolean elevatorIsFull() {
		return loadInElevator == maxLoad;
	}
	
	/*
	 * Gives signal to the Elevator-thread to move to the next floor. 
	 */
	private boolean okGo() {
		return getsOffAt[currentFloor] == 0 && noMoreTakers() && !empty();
	}
	
	/*
	 * Checks if there are no more Person-threads who want to enter the elevator at currentFloor.
	 */
	private boolean noMoreTakers() {
		if(elevatorDirection) {
			return (loadInElevator == maxLoad || goingUpOnFloor[currentFloor] == 0);		
		} else {
			return (loadInElevator == maxLoad || goingDownOnFloor[currentFloor] == 0);
		}
	}
	
	/*
	 * Checks if the Person-thread is not allowed to enter the elevator.
	 */
	private boolean dontEnter(int start, boolean direction) { 
		return start != currentFloor || elevatorIsFull() || doorsClosed || direction != elevatorDirection;
	}
	/*
	 * Checks if the entire simulation is empty.
	 */
	private boolean empty() {
		return totalLoad == 0;
	}
	
	private boolean optimizeRoute(boolean goingUp) {
		if(currentFloor == 0 || currentFloor == topFloor) {
			return false;
		} else if(goingUp) {
			return loadInElevator == 0 && noWaitAbove();
		} else {
			return loadInElevator == 0 && noWaitBelow();
		}
	}
	
	private boolean noWaitAbove() {
		for(int i = currentFloor + 1; i <= topFloor; i++) {
			if(loadOnFloor[i] != 0) {
				return false;
			}
		}
		return true;
	}
	
	private boolean noWaitBelow() {
		for(int i = currentFloor - 1; i >= 0; i--) {
			if(loadOnFloor[i] != 0) {
				return false;
			}
		}
		return true;
	}
	// mail till atf10rna@student.lu.se, marcus.s@telia.com
}

import javax.swing.*;

import java.awt.event.*;
import java.awt.*;


public class LiftView {

	private JFrame view;
	private FixedSizePanel entryPane,shaftPane;
	private FloorExit exitPane;
	private Basket basket;
	private static int FLOOR_HEIGHT = 100;
	private static int ENTRY_WIDTH = 300;
	private static int EXIT_WIDTH = 200;
	private static int SHAFT_WIDTH = 150;
	private static int NO_OF_FLOORS = 7;
	private static int MAX_LOAD = 4;
	private FloorEntry[] floorIn;

	public LiftView() {
		view = new JFrame("LiftView");
		view.getContentPane().setLayout(new BorderLayout());
		WindowListener l = new WindowAdapter() {
			public void windowClosing(WindowEvent e) {
				System.exit(0);
			}
		};
		view.addWindowListener(l);
		view.setResizable(false);
		entryPane = new FixedSizePanel(ENTRY_WIDTH,NO_OF_FLOORS*FLOOR_HEIGHT);
		entryPane.setLayout(new GridLayout(NO_OF_FLOORS,1));
		floorIn = new FloorEntry[NO_OF_FLOORS];
		for(int i=0;i<NO_OF_FLOORS;i++) {
			floorIn[NO_OF_FLOORS-i-1] = new FloorEntry(ENTRY_WIDTH,FLOOR_HEIGHT);
			entryPane.add(floorIn[NO_OF_FLOORS-i-1]);
		}
		view.getContentPane().add("West",entryPane);
		shaftPane = new FixedSizePanel(SHAFT_WIDTH,NO_OF_FLOORS*FLOOR_HEIGHT);
		shaftPane.setBackground(Color.LIGHT_GRAY);
		shaftPane.setLayout(null);
		view.getContentPane().add("Center",shaftPane);
		exitPane = new FloorExit(EXIT_WIDTH,NO_OF_FLOORS,FLOOR_HEIGHT);
		view.getContentPane().add("East",exitPane);
		basket = new Basket(SHAFT_WIDTH,NO_OF_FLOORS,FLOOR_HEIGHT,shaftPane);
		view.pack();
		view.setVisible(true);
	}

	public void drawLift(int floor, int load) {
		if (load<0 || load>MAX_LOAD) {
			throw new Error("Illegal load parameter to drawLift.");
		}
		if (floor<0 || floor>=NO_OF_FLOORS) {
			throw new Error("Illegal floor parameter to drawLift");
		}
		boolean animate = basket.getLoad()>load;
		basket.draw(floor,load);
		if (animate) {
			exitPane.animatePerson(floor); 
		}
	}

	public void drawLevel(int floor, int persons) {
		if (floor<0 || floor>=NO_OF_FLOORS) {
			throw new Error("Illegal floor in call to drawLevel.");
		}
		if (persons<0) {
			throw new Error("Negative number of persons in call to drawLevel.");
		}
		Thread.yield();
		floorIn[floor].draw(persons);
		Thread.yield();
	}

	public void moveLift(int here, int next) {
		if (here<0 || here>=NO_OF_FLOORS || next<0 || next>=NO_OF_FLOORS ||
				here==next) {
			throw new Error("Illegal parameters to moveLift.");
		}
		basket.moveBasket(here,next);
		try {
			Thread.sleep(200);
		} catch(InterruptedException e) { }
	}


	public static void main(String[] args) {
		int OFFICE_WORKERS = 20;
		LiftView lv = new LiftView();
		Monitor m = new Monitor(lv, NO_OF_FLOORS, MAX_LOAD);
		Elevator elevator = new Elevator(lv, m, NO_OF_FLOORS);
		elevator.start();
		Person[] persons = new Person[OFFICE_WORKERS];
		for(int i = 0; i < OFFICE_WORKERS; i++) {
			persons[i] = new Person(m, NO_OF_FLOORS);
			persons[i].start();
		}
	}

	private class FixedSizePanel extends JPanel {
		private static final long serialVersionUID = 1L;
		private Dimension dim;

		public FixedSizePanel(int w,int h) {
			dim = new Dimension(w,h);
			setSize(dim);
		}

		public Dimension getPreferredSize() {
			return dim;
		}
	}

	private class FloorEntry extends FixedSizePanel {
		private static final long serialVersionUID = 1L;
		private int width,height;
		private int waiting;

		public FloorEntry(int w,int h) {
			super(w,h);
			setBackground(Color.WHITE);
			height = h;
			width = w;
			waiting = 0;
		}

		public void draw(int w) {
			waiting = w;
			Thread.yield();
			repaint();
		}

		protected void paintComponent(Graphics g) {
			super.paintComponent(g);
			g.drawLine(0,height-1,width,height-1);
			for(int i=0;i<waiting;i++) {
				PersonDrawer.draw(g,ENTRY_WIDTH-(i+1)*35,height-5);
			}
		}
	}

	private class FloorExit extends FixedSizePanel {
		private static final long serialVersionUID = 1L;
		private int width,floorHeight,noOfFloors;
		private int animateX,animateY;

		public FloorExit(int w,int nof,int fh) {
			super(w,nof*fh);
			width = w;
			noOfFloors = nof;
			floorHeight = fh;
			setBackground(Color.WHITE);
			animateX = 0;
			animateY = 0;
		}

		protected void paintComponent(Graphics g) {
			super.paintComponent(g);
			for(int i=1;i<noOfFloors;i++) {
				g.drawLine(0,i*floorHeight-1,width,i*floorHeight-1);
			}
			if (animateY!=0) {
				PersonDrawer.draw(g,animateX,animateY);
			}
		}

		public void animatePerson(int floor) {
			animateY = (noOfFloors-floor)*floorHeight-5;
			for(animateX=0;animateX<width;animateX+=20) {
				repaint();
				try {
					Thread.sleep(100);
				} catch(InterruptedException e) { }
			}
			animateX = 0;
			animateY = 0;
			repaint();
		}
	}

	private class Basket extends FixedSizePanel {
		private static final long serialVersionUID = 1L;
		private int width,floorHeight,noOfFloors;
		private int INCREMENT = 3;
		private int load;

		public Basket(int w,int nof,int fh,FixedSizePanel shaft) {
			super(w-4,fh);
			width = w;
			noOfFloors = nof;
			floorHeight = fh;
			load = 0;
			setBackground(Color.GREEN);
			shaft.add(this);
			setLocation(2,(nof-1)*fh);
		}

		protected void paintComponent(Graphics g) {
			super.paintComponent(g);
			g.drawRect(0,0,width-5,floorHeight-1);
			for(int i=0;i<load;i++) {
				PersonDrawer.draw(g,i*35+5,floorHeight-5);
			}
		}

		private int floorOffset(int floor) {
			return (noOfFloors-floor-1)*floorHeight;
		}

		public int getLoad() {
			return load;
		}

		public void moveBasket(int from, int to) {
			int start = floorOffset(from);
			int stop = floorOffset(to);
			if (start<stop) {
				for(int y=start;y<stop;y+=INCREMENT) {
					setLocation(2,y);
					try {
						Thread.sleep(10);
					} catch(InterruptedException e) { }
				}
			} else {
				for(int y=start;y>stop;y-=INCREMENT) {
					setLocation(2,y);
					try {
						Thread.sleep(10);
					} catch(InterruptedException e) { }
				}
			}
			setLocation(2,stop);
		}

		public void draw(int f,int l) {
			load = l;
			setLocation(2,floorOffset(f));
			repaint();
		}
	}

}

class PersonDrawer {

	public static void draw(Graphics g,int x,int y) {
		g.drawLine(x,y,x+12,y-30);
		g.drawLine(x+12,y-30,x+24,y);
		g.drawLine(x+12,y-30,x+12,y-55);
		g.drawLine(x+12,y-55,x,y-35);
		g.drawLine(x+12,y-55,x+24,y-35);
		g.drawOval(x+5,y-70,15,15);
	}

	public static void erase(Graphics g,int x,int y) {
		Color c = g.getColor();
		g.setColor(Color.WHITE);
		draw(g,x,y);
		g.setColor(c);
	}

}
```

[^pun_footnote]: Forgive the pun.
