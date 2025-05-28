
#include <stdio.h>
#include <stdlib.h>
#include <stdbool.h>
#include <assert.h>
#include <windows.h>
#include <conio.h>

#define ROWS 21
#define COLS 57

#define BUS_STOPS 16
#define BUS_LINES 6
#define MAX_LINE_SIZE BUS_STOPS
#define BUSES_PER_LINE 2
#define BUSES_TOTAL (BUSES_PER_LINE * BUS_LINES)
#define PASSENGER_MAX_LUGGAGES 2
#define BUS_MAX_LUGGAGES 8
#define BUS_MAX_PASSENGERS 8
#define NULL_PASSENGER_ID -1
#define TICKS_PER_BUS_MOVEMENT 2
#define PASSENGERS_IN_QUEUE 15
#define SECS_PER_TICK 0.5
#define PSG_QUEUE_MAX_WAIT_TICKS 1
#define FPS 2
#define FRAME_MS (1/FPS*1000)

#define ROAD_CHAR ' '
#define OBSTACLE_CHAR '#'

#define FOREGROUND_WHITE (FOREGROUND_RED | FOREGROUND_GREEN | FOREGROUND_BLUE | FOREGROUND_INTENSITY)
#define OBSTACLE_ATTRIBUTE (FOREGROUND_WHITE | BACKGROUND_BLUE)
#define ROAD_ATTRIBUTE (BACKGROUND_RED | BACKGROUND_GREEN | BACKGROUND_BLUE)
#define STOP_ATTRIBUTE ROAD_ATTRIBUTE
#define SELECTED_CELL_ATTR (BACKGROUND_GREEN | BACKGROUND_INTENSITY)

typedef struct Passenger Passenger;
typedef struct PsgNode PsgNode;
typedef struct SimulationData SimulationData;

void select_next(SimulationData* sim_data);
void select_prev(SimulationData* sim_data);

typedef enum BusDirInLine {
	DIR_FIRST_TO_LAST,
	DIR_LAST_TO_FIRST
} BusDirInLine;

typedef enum SelectionType {
	SELECT_BUS_STOP,
	SELECT_BUS
} SelectionType;

typedef struct Vector2 {
	int x, y;
} Vector2;

typedef struct Luggage {
	int id;
	int owner_id;
} Luggage;

typedef struct LuggageList {
	Luggage* items;
	size_t count;
	size_t capacity;
} LuggageList;

typedef struct Passenger {
	int id;
	LuggageList luggages;
	char starting_stop, target_stop;
	char line;
} Passenger;

typedef struct PsgNode {
	Passenger value;
	PsgNode* next;
} PsgNode;

typedef struct PsgQueue {
	PsgNode* rear;
	PsgNode* front;
} PsgQueue;

typedef struct BusLine {
	char stops[MAX_LINE_SIZE];
	size_t stop_count;
} BusLine;

typedef struct BusStop {
	char name;
	Vector2 position;
	PsgQueue passengers;
	size_t passenger_count;
} BusStop;

typedef struct Bus {
	Vector2 position;
	Vector2 move_direction;
	BusDirInLine line_direction;
	Passenger passengers[BUS_MAX_PASSENGERS];
	LuggageList luggages;
	size_t passenger_count;
	BusLine line;
	BusStop* arrived_at_stop;
	size_t ticks_to_move;
	size_t target_stop_idx_in_sim;
} Bus;

typedef struct SimulationData {
	BusStop stops[BUS_STOPS];
	Bus buses[BUSES_TOTAL];
	size_t ticks_passed;
	float time_to_next_tick;
	bool is_paused;
	size_t psg_wait_ticks;
	PsgQueue psg_queue;
	size_t waiting_psg_count;
	size_t travelling_psg_count;
	HANDLE h_in;
	DWORD prev_mode;
	void* selection;
	SelectionType selection_type;
	Vector2 selection_pos;
} SimulationData;

HANDLE h_out, h_in;
DWORD prev_h_in_mode;

static const char grid[ROWS][COLS + 1] = {
	"#########################################################",
	"#A      #####B            #####C                 D#######",
	"# ########### ########### ##### ##### ########### #######",
	"# ########### ########### ##### ##### ########### #######",
	"# ########### ########### ##### ##### ########### #######",
	"# ########### ########### ##### ##### ########### #######",
	"# ########### ########### ##### ##### ########### #######",
	"#E           F                 G#####            H      #",
	"# ########### ##### ########### ##### ##### ########### #",
	"# ########### ##### ########### ##### ##### ########### #",
	"# ########### ##### ########### ##### ##### ########### #",
	"# ########### ##### ########### ##### ##### ########### #",
	"# ########### ##### ########### ##### ##### ########### #",
	"#I           J                 K           L#####       #",
	"####### ##### ##### ########### ##### ##### ##### #######",
	"####### ##### ##### ########### ##### ##### ##### #######",
	"####### ##### ##### ########### ##### ##### ##### #######",
	"####### ##### ##### ########### ##### ##### ##### #######",
	"####### ##### ##### ########### ##### ##### ##### #######",
	"#      M     N      #####      O           P            #",
	"#########################################################",
};

static const char bus_lines[BUS_LINES][MAX_LINE_SIZE + 1] = {
	"AEIJKL",
	"BFEIJNM",
	"CGKJFEA",
	"DCGKJNM",
	"LPOKGCDH",
	"MNJFGKOP"
};

PsgNode* psgnode_alloc(Passenger psg) {
	PsgNode* node = malloc(sizeof(PsgNode)); assert(node != NULL);
	node->value = psg;
	node->next = NULL;
	return node;
}

PsgQueue psgqueue_create() {
	return (PsgQueue) {
		.rear = NULL,
		.front = NULL
	};
}

void psgqueue_destroy(PsgQueue* queue) {
	while (queue->front) {
		PsgNode* prev_front = queue->front;
		queue->front = prev_front->next;
		free(prev_front);
	}
	queue->rear = NULL;
}

void psgqueue_enqueue(PsgQueue* queue, Passenger psg) {
	PsgNode* node = psgnode_alloc(psg);
	if (queue->rear == NULL) {
		queue->front = node;
		queue->rear = node;
	}
	else {
		PsgNode* prev_rear = queue->rear;
		queue->rear = node;
		prev_rear->next = queue->rear;
	}
}

Passenger psgqueue_dequeue(PsgQueue* queue) {
	if (queue->front == NULL) {
		return (Passenger) { .id = NULL_PASSENGER_ID };
	}
	else {
		Passenger psg = queue->front->value;
		PsgNode* prev_front = queue->front;
		queue->front = prev_front->next;
		free(prev_front);
		return psg;
	}
}

void psgqueue_remove_node(PsgQueue* queue, PsgNode* node) {
	PsgNode* prev = NULL;
	PsgNode* curr = queue->front;
	while (curr) {
		if (curr == node) {
			if (prev == NULL) {
				queue->front = curr->next;
			}
			else if (curr->next == NULL) {
				queue->rear = prev;
				prev->next = NULL;
			}
			else {
				prev->next = curr->next;
			}
			free(curr);
			return;
		}
		curr = curr->next;
	}
}

LuggageList luggagelist_create() {
	return (LuggageList) {
		.items = NULL,
		.count = 0,
		.capacity = 0
	};
}

void luggagelist_destroy(LuggageList* list) {
	free(list->items);
	list->items = NULL;
	list->capacity = 0;
	list->count = 0;
}

void luggagelist_append(LuggageList* list, Luggage luggage) {
	if (list->capacity == 0) {
		list->capacity = list->count * 2 + 4;
		list->items = malloc(list->capacity * sizeof(Luggage)); assert(list->items != NULL);
	}
	else if (list->count >= list->capacity) {
		list->capacity = list->count * 2;
		Luggage* new_items = realloc(list->items, list->capacity * sizeof(Luggage)); assert(new_items != NULL);
		list->items = new_items;
	}
	list->items[list->count++] = luggage;
}

void luggagelist_delete_id(LuggageList* list, int id) {
	size_t idx_to_delete = list->count;
	for (size_t i = 0; i < list->count; i++) {
		if (list->items[i].id == id) {
			idx_to_delete = i;
			break;
		}
	}
	for (size_t i = idx_to_delete; i < list->count - 1; i++) {
		list->items[i] = list->items[i + 1];
	}
	list->count--;
}

Vector2 dir_from_to_stop(BusStop from, BusStop to) {
	int x_diff = to.position.x - from.position.x;
	int y_diff = to.position.y - from.position.y;
	return (Vector2) {
		(x_diff > 0) - (x_diff < 0),
		(y_diff > 0) - (y_diff < 0)
	};
}

Passenger psg_generate_random(SimulationData* sim_data) {
	size_t target_bus = rand() % BUSES_TOTAL;
	BusLine target_line = sim_data->buses[target_bus].line;

	int id = rand() % 1000;
	size_t starting_stop_idx = rand() % target_line.stop_count;
	size_t target_stop_idx = starting_stop_idx;

	while (target_stop_idx == starting_stop_idx) {
		target_stop_idx = rand() % target_line.stop_count;
	}

	LuggageList luggages = luggagelist_create();
	size_t luggage_count = rand() % (PASSENGER_MAX_LUGGAGES + 1);
	for (size_t i = 0; i < luggage_count; i++) {
		luggagelist_append(&luggages, (Luggage) {
			.id = rand() % 1000,
			.owner_id = id,
		});
	}

	return (Passenger) {
		.id = id,
		.luggages = luggages,
		.starting_stop = target_line.stops[starting_stop_idx],
		.target_stop = target_line.stops[target_stop_idx],
		.line = target_line.stops[0]
	};
}

Bus bus_create(BusStop* all_stops, BusLine line, BusDirInLine line_dir) {
	size_t stop_idx_in_line = line_dir == DIR_FIRST_TO_LAST ? 0 : line.stop_count - 1;
	size_t stop_idx_in_city = line.stops[stop_idx_in_line] - 'A';
	return (Bus){
		.position = all_stops[stop_idx_in_city].position,
		.line_direction = line_dir,
		.passenger_count = 0,
		.luggages = luggagelist_create(),
		.line = line,
		.arrived_at_stop = &all_stops[stop_idx_in_city],
		.ticks_to_move = TICKS_PER_BUS_MOVEMENT
	};
}

SimulationData* simdata_alloc() {
	SimulationData* data = malloc(sizeof(SimulationData)); assert(data != NULL);
	data->ticks_passed = 0;
	data->is_paused = true;
	data->psg_queue = psgqueue_create();
	data->psg_wait_ticks = rand() % (PSG_QUEUE_MAX_WAIT_TICKS + 1);
	data->waiting_psg_count = 0;
	data->travelling_psg_count = 0;
	data->selection = NULL;

	for (size_t row = 0; row < ROWS; row++) {
		for (size_t col = 0; col < COLS; col++) {
			char cell = grid[row][col];
			if (cell >= 'A' && cell < 'A' + BUS_STOPS) {
				size_t bus_stop_idx = cell - 'A';
				data->stops[bus_stop_idx] = (BusStop){
					.name = cell,
					.position = (Vector2){ col, row },
					.passengers = psgqueue_create(),
					.passenger_count = 0
				};
			}
		}
	}

	for (size_t i = 0; i < BUS_LINES; i++) {
		size_t line_len = strnlen_s(bus_lines[i], MAX_LINE_SIZE);
		BusLine line;
		line.stop_count = line_len;
		strncpy_s(line.stops, MAX_LINE_SIZE, bus_lines[i], line_len);
		data->buses[i * 2] = bus_create(data->stops, line, DIR_FIRST_TO_LAST);
		data->buses[i * 2 + 1] = bus_create(data->stops, line, DIR_LAST_TO_FIRST);
	}

	for (size_t i = 0; i < PASSENGERS_IN_QUEUE; i++) {
		psgqueue_enqueue(&data->psg_queue, psg_generate_random(data));
	}

	select_next(data);
	select_prev(data);
	return data;
}

void simdata_destroy(SimulationData* sim_data) {
	PsgNode* node = sim_data->psg_queue.front; 
	while (node) {
		luggagelist_destroy(&node->value.luggages);
		node = node->next;
	}
	psgqueue_destroy(&sim_data->psg_queue);

	for (size_t i = 0; i < BUSES_TOTAL; i++) {
		luggagelist_destroy(&sim_data->buses[i].luggages);
	}

	for (size_t i = 0; i < BUS_STOPS; i++) {
		node = sim_data->stops[i].passengers.front;
		while (node) {
			luggagelist_destroy(&node->value.luggages);
			node = node->next;
		}
		psgqueue_destroy(&sim_data->stops[i].passengers);
	}
}

void simdata_bus_advance_tick(SimulationData* sim_data, Bus* bus) {
	if (bus->arrived_at_stop) {
		for (int i = bus->passenger_count - 1; i >= 0; i--) {
			if (bus->passengers[i].target_stop != bus->arrived_at_stop->name)
				continue;
			for (size_t k = 0; k < bus->luggages.count; k++) {
				if (bus->luggages.items[k].owner_id == bus->passengers[i].id) {
					luggagelist_delete_id(&bus->luggages, bus->luggages.items[k].id);
					return;
				}
			}

			luggagelist_destroy(&bus->passengers[i].luggages);
			for (int k = i; k < bus->passenger_count - 1; k++) {
				bus->passengers[k] = bus->passengers[k + 1];
			}
			bus->passenger_count--;
			sim_data->travelling_psg_count--;
		}

		PsgNode* psg_node = bus->arrived_at_stop->passengers.front;
		while (bus->passenger_count < BUS_MAX_PASSENGERS && psg_node != NULL) {
			if (BUS_MAX_LUGGAGES - bus->luggages.count < psg_node->value.luggages.count) {
				psg_node = psg_node->next;
				continue;
			}

			bool psg_should_get_on = false;
			for (size_t i = 0; i < bus->line.stop_count; i++) {
				if (bus->line.stops[i] == psg_node->value.target_stop) {
					psg_should_get_on = true;
					break;
				}
			}
			
			if (!psg_should_get_on) {
				psg_node = psg_node->next;
				continue;
			}

			for (size_t i = 0; i < psg_node->value.luggages.count; i++) {
				Luggage luggage = psg_node->value.luggages.items[i];
				luggagelist_append(&bus->luggages, luggage);
				luggagelist_delete_id(&psg_node->value.luggages, luggage.id);
				return;
			}

			PsgQueue new_psgq = psgqueue_create();
			PsgNode* new_curr = bus->arrived_at_stop->passengers.front;
			while (new_curr) {
				if (new_curr != psg_node) {
					psgqueue_enqueue(&new_psgq, new_curr->value);
				}
				new_curr = new_curr->next;
			}
			bus->arrived_at_stop->passengers = new_psgq;
			bus->arrived_at_stop->passenger_count--;
			bus->passengers[bus->passenger_count++] = psg_node->value;
			sim_data->waiting_psg_count--;
			sim_data->travelling_psg_count++;
			break;
		}

		size_t next_stop_idx_in_sim;
		char* curr_stop_in_stops = strchr(bus->line.stops, bus->arrived_at_stop->name);
		char* last_stop_in_stops = &bus->line.stops[bus->line.stop_count - 1];
		if (bus->line_direction == DIR_FIRST_TO_LAST) {
			if (curr_stop_in_stops == last_stop_in_stops) {
				bus->line_direction = DIR_LAST_TO_FIRST;
				next_stop_idx_in_sim = *(curr_stop_in_stops - 1) - 'A';
			}
			else {
				next_stop_idx_in_sim = *(curr_stop_in_stops + 1) - 'A';
			}
		}
		else {
			if (curr_stop_in_stops == &bus->line.stops[0]) {
				bus->line_direction = DIR_FIRST_TO_LAST;
				next_stop_idx_in_sim = *(curr_stop_in_stops + 1) - 'A';
			}
			else {
				next_stop_idx_in_sim = *(curr_stop_in_stops - 1) - 'A';
			}
		}
		bus->target_stop_idx_in_sim = next_stop_idx_in_sim;
		bus->move_direction = dir_from_to_stop(*bus->arrived_at_stop, sim_data->stops[next_stop_idx_in_sim]);
		bus->arrived_at_stop = NULL;
		bus->ticks_to_move = TICKS_PER_BUS_MOVEMENT;
		return;
	}

	bus->ticks_to_move--;
	if (bus->ticks_to_move > 0)
		return;
	bus->ticks_to_move = TICKS_PER_BUS_MOVEMENT;

	bus->position.x += bus->move_direction.x;
	bus->position.y += bus->move_direction.y;
	BusStop* target_stop = &sim_data->stops[bus->target_stop_idx_in_sim];
	if (target_stop->position.x == bus->position.x && target_stop->position.y == bus->position.y) {
		bus->arrived_at_stop = target_stop;
	}
}

void simdata_advance_tick(SimulationData* sim_data) {
	sim_data->ticks_passed++;
	if (sim_data->psg_wait_ticks == 0) {
		Passenger psg = psgqueue_dequeue(&sim_data->psg_queue);
		BusStop* starting_stop = &sim_data->stops[psg.starting_stop - 'A'];
		psgqueue_enqueue(&starting_stop->passengers, psg);
		starting_stop->passenger_count++;
		sim_data->waiting_psg_count++;
		psgqueue_enqueue(&sim_data->psg_queue, psg_generate_random(sim_data));
		sim_data->psg_wait_ticks = rand() % (PSG_QUEUE_MAX_WAIT_TICKS + 1);
	}
	else {
		sim_data->psg_wait_ticks--;
	}

	for (size_t i = 0; i < BUSES_TOTAL; i++) {
		simdata_bus_advance_tick(sim_data, &sim_data->buses[i]);
	}
}

bool simdata_advance_secs(SimulationData* sim_data, float secs_passed) {
	if (sim_data->is_paused)
		return false;

	sim_data->time_to_next_tick -= secs_passed;
	if (sim_data->time_to_next_tick > 0)
		return false;
	sim_data->time_to_next_tick = SECS_PER_TICK;

	simdata_advance_tick(sim_data);
	return true;
}

void clear_screen() {
	HANDLE h_console = GetStdHandle(STD_OUTPUT_HANDLE);

	CONSOLE_SCREEN_BUFFER_INFO csbi;
	SMALL_RECT scroll_rect;
	COORD scroll_target;
	CHAR_INFO fill;

	if (!GetConsoleScreenBufferInfo(h_console, &csbi))
	{
		return;
	}

	scroll_rect.Left = 0;
	scroll_rect.Top = 0;
	scroll_rect.Right = csbi.dwSize.X;
	scroll_rect.Bottom = csbi.dwSize.Y;

	scroll_target.X = 0;
	scroll_target.Y = (SHORT)(0 - csbi.dwSize.Y);

	fill.Char.UnicodeChar = TEXT(' ');
	fill.Attributes = csbi.wAttributes;

	ScrollConsoleScreenBuffer(h_console, &scroll_rect, NULL, scroll_target, &fill);

	csbi.dwCursorPosition.X = 0;
	csbi.dwCursorPosition.Y = 0;

	SetConsoleCursorPosition(h_console, csbi.dwCursorPosition);
}

void place_char_and_attr(int x, int y, char c, int attr) {
	HANDLE h_console = GetStdHandle(STD_OUTPUT_HANDLE);
	LONG chars_written;
	WriteConsoleOutputCharacterA(h_console, &c, 1, (COORD) { x, y }, & chars_written);
	WriteConsoleOutputAttribute(h_console, &attr, 1, (COORD) { x, y }, & chars_written);
}

void place_char(int x, int y, char c) {
	HANDLE h_console = GetStdHandle(STD_OUTPUT_HANDLE);
	LONG chars_written;
	WriteConsoleOutputCharacterA(h_console, &c, 1, (COORD) { x, y }, & chars_written);
}

void place_attr(int x, int y, int attr) {
	HANDLE h_console = GetStdHandle(STD_OUTPUT_HANDLE);
	LONG chars_written;
	WriteConsoleOutputAttribute(h_console, &attr, 1, (COORD) { x, y }, & chars_written);
}

void place_number(int x, int y, int n) {
	char buffer[256];
	sprintf_s(buffer, 256, "%d", n);
	size_t len = strnlen_s(buffer, 256);

	for (size_t i = 0; i < len; i++) {
		place_char(x + i, y, buffer[i]);
	}
}

void move_cursor(int x, int y) {
	HANDLE h_console = GetStdHandle(STD_OUTPUT_HANDLE);
	SetConsoleCursorPosition(h_console, (COORD) { x, y });
}

void draw_time(SimulationData* sim_data, size_t gap) {
	move_cursor(COLS + gap, 0);
	printf("Time: %7zu", sim_data->ticks_passed);
	if (sim_data->is_paused) {
		printf(" (Paused)");
	}
}

void draw_map(SimulationData* sim_data) {
	for (int x = 1; x <= COLS; x++) {
		place_char_and_attr(x, 0, '0' + (x % 10), FOREGROUND_WHITE);
	}
	for (int y = 1; y <= ROWS; y++) {
		place_char_and_attr(0, y, '0' + (y % 10), FOREGROUND_WHITE);
	}

	static const int xOff = 1;
	static const int yOff = 1;

	for (size_t row = 0; row < ROWS; row++) {
		for (size_t col = 0; col < COLS; col++) {
			char cell = grid[row][col];
			int attr;
			switch (cell) {
			case ROAD_CHAR: attr = ROAD_ATTRIBUTE; break;
			case OBSTACLE_CHAR: attr = OBSTACLE_ATTRIBUTE; break;
			default: attr = STOP_ATTRIBUTE; break;
			}
			place_char_and_attr(xOff + col, yOff + row, cell, attr);
		}
	}

	for (size_t i = 0; i < BUS_STOPS; i++) {
		BusStop s = sim_data->stops[i];
		place_char(xOff + s.position.x, yOff + s.position.y, s.name);
		place_number(xOff + s.position.x + 1, yOff + s.position.y + 1, s.passenger_count);

		if (sim_data->selection == &sim_data->stops[i])
			place_attr(xOff + s.position.x, yOff + s.position.y, SELECTED_CELL_ATTR);
	}



	for (size_t i = 0; i < BUSES_TOTAL; i++) {
		Bus b = sim_data->buses[i];
		place_number(xOff + b.position.x, yOff + b.position.y, b.passenger_count);

		if (sim_data->selection == &sim_data->buses[i])
			place_attr(xOff + b.position.x, yOff + b.position.y, SELECTED_CELL_ATTR);;
	}
}

void draw_new_psg(SimulationData* sim_data, size_t gap) {
	move_cursor(COLS + gap, 2);
	printf("New Passengers");
	move_cursor(COLS + gap, 3);
	printf("---------------");
	move_cursor(COLS + gap - 2, 4);
	printf("> ");
	
	PsgNode* psg_node = sim_data->psg_queue.front;
	int x = COLS + gap + PASSENGERS_IN_QUEUE - 1;
	place_char(x + 2, 4, '>');
	while (psg_node) {
		place_char(x, 4, psg_node->value.starting_stop);
		psg_node = psg_node->next;
		x--;
	}

	move_cursor(COLS + gap, 5);
	printf("---------------");
}

void draw_stats(SimulationData* sim_data, int gap) {
	move_cursor(COLS + gap, 7);
	printf("Waiting     : %3d", sim_data->waiting_psg_count);
	move_cursor(COLS + gap, 8);
	printf("Travelling  : %3d", sim_data->travelling_psg_count);
	move_cursor(COLS + gap, 9);
	int bus_fullness = sim_data->travelling_psg_count * 100 / (BUSES_TOTAL * BUS_MAX_PASSENGERS);
	printf("Bus Fullness: %3d%%", bus_fullness);
}

void draw_selection(SimulationData* sim_data, int gap) {
	if (sim_data->selection == NULL)
		return;

	if (sim_data->selection_type == SELECT_BUS) {
		Bus* b = (Bus*)sim_data->selection;
		char bus_letter = b->line.stops[0];
		char bus_no = b->line_direction == DIR_FIRST_TO_LAST ? '1' : '2';

		move_cursor(COLS + gap, 11);
		printf("Bus [%c%c] Passengers:", bus_letter, bus_no);

		int y = 12;
		for (size_t i = 0; i < b->passenger_count; i++) {
			Passenger p = b->passengers[i];
			LuggageList psg_luggages = luggagelist_create();
			for (size_t k = 0; k < b->luggages.count; k++) {
				if (b->luggages.items[k].owner_id == p.id) {
					luggagelist_append(&psg_luggages, b->luggages.items[k]);
				}
			}

			move_cursor(COLS + gap, y);
			printf("%3d: %c-%c", p.id, p.starting_stop, p.target_stop);
			switch (psg_luggages.count) {
			case 0: printf(" (L:-)"); break;
			case 1: printf(" (L:%d)", psg_luggages.items[0].id); break;
			case 2: printf(" (L:%d,%d)", psg_luggages.items[0].id, psg_luggages.items[1].id); break;
			}
			y++;
			luggagelist_destroy(&psg_luggages);
		}

		y = 21;
		move_cursor(COLS + gap, y);
		printf("   Bus[A2] Luggages:");
		move_cursor(COLS + gap + 22, y + 1);
		printf("+-----+");

		for (int i = 0; i < 8; i++) {
			move_cursor(COLS + gap + 22, y - i);
			printf("|     |");
		}

		for (int i = 0; i < b->luggages.count; i++) {
			move_cursor(COLS + gap + 24, y - i);
			printf("%3d", b->luggages.items[i].id);
		}
	}
	else {
		BusStop* s = (BusStop*)sim_data->selection;
		move_cursor(COLS + gap, 11);
		printf("Bus Stop [%c] Passengers:\n", s->name);
		int y = 12;
		PsgNode* node = s->passengers.front;
		while (node) {
			move_cursor(COLS + gap, y);
			printf("%3d: Line%c %c-%c", node->value.id, node->value.line, node->value.starting_stop, node->value.target_stop);
			switch (node->value.luggages.count)
			{
			case 0: printf(" (L:-)"); break;
			case 1: printf(" (L:%d)", node->value.luggages.items[0].id); break;
			case 2: printf(" (L:%d,%d)", node->value.luggages.items[0].id, node->value.luggages.items[1].id); break;
			}
			node = node->next;
			y++;
		}
	}
}

void draw_screen(SimulationData* sim_data) {
	clear_screen();
	draw_map(sim_data);
	draw_time(sim_data, 4);
	draw_new_psg(sim_data, 4);
	draw_stats(sim_data, 2);
	draw_selection(sim_data, 2);
}

void select_next(SimulationData* sim_data) {
	Vector2 best_pos = { INT_MAX, INT_MAX };
	void* best_item = NULL;
	SelectionType best_type;

	for (size_t i = 0; i < BUSES_TOTAL; i++) {
		Bus* b = &sim_data->buses[i];

		if ((b->position.y > sim_data->selection_pos.y) ||
			(b->position.y == sim_data->selection_pos.y && b->position.x > sim_data->selection_pos.x)) {

			if ((b->position.y < best_pos.y) ||
				(b->position.y == best_pos.y && b->position.x < best_pos.x)) {
				best_pos = b->position;
				best_item = b;
				best_type = SELECT_BUS;
			}
		}
	}

	for (size_t i = 0; i < BUS_STOPS; i++) {
		BusStop* s = &sim_data->stops[i];

		if ((s->position.y > sim_data->selection_pos.y) ||
			(s->position.y == sim_data->selection_pos.y && s->position.x > sim_data->selection_pos.x)) {

			if ((s->position.y < best_pos.y) ||
				(s->position.y == best_pos.y && s->position.x < best_pos.x)) {
				best_pos = s->position;
				best_item = s;
				best_type = SELECT_BUS_STOP;
			}
		}
	}

	if (best_item != NULL) {
		sim_data->selection = best_item;
		sim_data->selection_pos = best_pos;
		sim_data->selection_type = best_type;
	}
}

void select_prev(SimulationData* sim_data) {
	Vector2 best_pos = { -1, -1 };
	void* best_item = NULL;
	SelectionType best_type;

	for (size_t i = 0; i < BUSES_TOTAL; i++) {
		Bus* b = &sim_data->buses[i];

		if ((b->position.y < sim_data->selection_pos.y) ||
			(b->position.y == sim_data->selection_pos.y && b->position.x < sim_data->selection_pos.x)) {

			if ((b->position.y > best_pos.y) ||
				(b->position.y == best_pos.y && b->position.x > best_pos.x)) {
				best_pos = b->position;
				best_item = b;
				best_type = SELECT_BUS;
			}
		}
	}

	for (size_t i = 0; i < BUS_STOPS; i++) {
		BusStop* s = &sim_data->stops[i];

		if ((s->position.y < sim_data->selection_pos.y) ||
			(s->position.y == sim_data->selection_pos.y && s->position.x < sim_data->selection_pos.x)) {

			if ((s->position.y > best_pos.y) ||
				(s->position.y == best_pos.y && s->position.x > best_pos.x)) {
				best_pos = s->position;
				best_item = s;
				best_type = SELECT_BUS_STOP;
			}
		}
	}

	if (best_item != NULL) {
		sim_data->selection = best_item;
		sim_data->selection_pos = best_pos;
		sim_data->selection_type = best_type;
	}
}


int main() {
	SimulationData* sim_data = simdata_alloc();

	ULONGLONG last_update = GetTickCount64();
	draw_screen(sim_data);

	while (true) {
		if (_kbhit()) {
			switch (_getch()) {
			case 'r':
			case 'R':
				sim_data->is_paused = !sim_data->is_paused;
				break;
			case 'q':
			case 'Q':
				select_prev(sim_data);
				break;
			case 'w':
			case 'W':
				select_next(sim_data);
				break;
			}
			draw_screen(sim_data);
		}

		ULONGLONG curr = GetTickCount64();
		if (simdata_advance_secs(sim_data, (curr - last_update) / 1000.0)) {
			draw_screen(sim_data);
		}

		last_update = curr;
	}
	simdata_destroy(sim_data);
}
