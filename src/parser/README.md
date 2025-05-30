## Building instructions

```cargo build --release```



## Running tests
```cargo test --release```


## Running in rust
For example add this main.rs inside the src/ folder:

```Rust
use ahash::AHashMap;
use memmap2::MmapOptions;
use parser::first_pass::parser_settings::ParserInputs;
use parser::parse_demo::Parser;
use parser::second_pass::parser_settings::create_huffman_lookup_table;
use std::fs::File;

fn main() {
    let path_to_demo = "test_demo.dem";
    let huf = create_huffman_lookup_table();

    let settings = ParserInputs {
        wanted_players: vec![],
        real_name_to_og_name: AHashMap::default(),
        wanted_player_props: vec!["X".to_string()],
        wanted_events: vec!["player_death".to_string()],
        wanted_other_props: vec![],
        parse_ents: true,
        wanted_ticks: vec![],
        parse_projectiles: false,
        only_header: false,
        list_props: false,
        only_convars: false,
        huffman_lookup_table: &huf,
        fallback_bytes: None,
        wanted_prop_states: AHashMap::default(),
        order_by_steamid: false,
    };
    let mut ds = Parser::new(settings, parser::parse_demo::ParsingMode::Normal);
    let file = File::open(path_to_demo).unwrap();
    let mmap = unsafe { MmapOptions::new().map(&file).unwrap() };
    let _output = ds.parse_demo(&mmap).unwrap();
}
```
Then we can run with ```cargo run --release```


Another useful command is ```cargo run --release -- -debug true``` which will print all decoded entity updates according to your filter in "debug_inspect()" function in second_pass/entities.rs



Note that the library is (at least currently) not meant to be used directly from rust and this is just to develop the library without having to build bindings during development.







### Notes on parser


The outer loop trough the file is very simple and can be found in parsing/parser.rs.

1. Read what command the next bytes will be.
2. Read what tick the next command is.
3. Read how many bytes the next command is.

Repeat above until DEM_Stop command
```Rust
    pub fn start(&mut self){
        loop {
            let cmd = self.read_varint()?;
            let tick = self.read_varint()?;
            let size = self.read_varint()?;

            let bytes = self.bytes[cur_pos..cur_pos + size]
            cur_pos += size

            match demo_cmd_type_from_int(msg_type as i32) {
                DEM_Packet => self.parse_packet(&bytes),
                DEM_FileHeader => self.parse_header(&bytes),
                DEM_FileInfo => self.parse_file_info(&bytes),
                DEM_SendTables => self.parse_classes(&bytes),
                DEM_ClassInfo => self.parse_class_info(&bytes),
                DEM_SignonPacket => self.parse_packet(&bytes),
                DEM_FullPacket => self.parse_full_packet(&bytes),
                DEM_UserCmd => self.parse_user_command_cmd(&bytes),
                DEM_StringTables => self.parse_stringtable_cmd(&bytes),
                DEM_Stop => break,
                _ => {},
            };
        }
    }
```
(slightly simplified from source code)


### Dem_packet
DEM_Packet is the most common CMD. The DEM_Packet is decoded similarly to how the outer loop works:

1. Read what msg type the next bytes are.
2. Read how many bytes the next message is.

I think there is no signal for stop, you just read the bitstream as long as there are >= 8 bits remaining.

```Rust
let mut bitreader = Bitreader::new(&bytes);

while bitreader.reader.has_bits_remaining(8) {

    let msg_type = bitreader.read_u_bit_var().unwrap();
    let size = bitreader.read_varint().unwrap();
    let msg_bytes = bitreader.read_n_bytes(size as usize).unwrap();

    match netmessage_type_from_int(msg_type as i32) {
        svc_PacketEntities => self.parse_packet_ents(&msg_bytes),
        svc_CreateStringTable => self.parse_create_stringtable(&msg_bytes),
        svc_UpdateStringTable => self.update_string_table(&msg_bytes),
        GE_Source1LegacyGameEventList => self.parse_game_event_map(&msg_bytes),
        GE_Source1LegacyGameEvent => self.parse_event(&msg_bytes),
        // ... some extra left out
        _ => {},
    };
```
(slightly simplified from source code)
 
The main message types are:

- svc_CreateStringTable
- svc_UpdateStringTable
- svc_PacketEntities
- GE_Source1LegacyGameEventList
- GE_Source1LegacyGameEvent

There are also some rare other message types that can be found in parsing/parser.rs


#### Stringtables (both create and update)
Stringtables are messages that contains suprise surprise string data. Most of this data is not interesting for demo parsing, but baselines for entities pass trough here. These are the "default" values for entities. For example a players default health is 100?. 
"Userinfo" Also comes trough here, but with the source2 format it seems to be included in packet-entities and doesn't seem interesting anymore.

#### GE_Source1LegacyGameEventList
Includes needed data for parsing game events. This message should come before the first game event. It tells you how the game events should be decoded. If this message is lost then I think decoding game events is not possible?

#### GE_Source1LegacyGameEvent
Typical game events. Triggered when interesting things happen in the game, like when a weapon is fired or a round is over. Events are just key value pairs. These events can be parsed seperately from packet_entities (assuming you dont want to add extra custom values into the events).

#### svc_PacketEntities
The majority of data in the demo. All the data relating to entities. For example every players every coordinate, health, viewangles and you name it. If you can see a value in a replay then it probably comes from here.  

Something to note is that packet entities only send changes in values. If a player is standing still then the players coordinates are not updated during that tick. This means that a value at tick 5000 may have been set at tick 3542 (when the player last moved) so you can't just parse the ticks that you are interested in, but also ticks before that. In theory it is possible to start a tick and parse ticks backwards until you find the most recent update, but this idea is very messy to implement.

This part combined with command "DEM_SendTables" are by far the most comlicated parts of the demo. Getting these right is way harder than the rest of the demo parsing. If you want to try parsing the demo I would recommend by starting with game events and then move on to these.



### Other stuff

The demo has 2 headers. First header 16 bytes and is just demo magic + how long file is expected to be. The other header is the message DEM_FileHeader and has some more info like what map was played.

