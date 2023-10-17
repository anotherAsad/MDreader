## Use DRAMSim3 in gem5

1. Clone the repository of DRAMSim3 in `/gem5/ext/dramsim3`:

   1. `cd /gem5/ext/dramsim3`
   2. `git clone https://github.com/umd-memsys/DRAMsim3.git`

2. Back to `/gem5` and compile it:

   1. `cd /gem5`
   2. Within the root of the gem5 directory, gem5 can be built with SCons using: `scons build/{ISA}/gem5.{variant} -j {cpus}`, say we want to build X86 in 32 threads: `scons build/X86/gem5.opt -j 32`

3. An example in running executable file (compiled from C++) is in `/gem5/configs/learning_gem5/part1/simple.py`

   here:

   ```python
   binary = os.path.join(
       thispath,
       "../../../",
       "tests/test-progs/hello/bin/x86/linux/hello",
   )
   ```

   change to an executable file, say 

   ```python
   binary = os.path.join(
   '/test/simple_hash'
   )
   ```

   and here I wrote a simple hash in C++, named `simple_hash.cpp`

   ```c++
   #include <iostream>
   #include <vector>
   
   class SimpleHash {
   private:
       std::vector<int> table;
       int size;
       
   public:
       SimpleHash(int s) : table(s, -1), size(s) {}
       
       int hashFunction(int key) {
           return key % size;
       }
       
       void insert(int key) {
           int index = hashFunction(key);
           if (table[index] == -1) {
               table[index] = key;
           } else {
               std::cout << "Collision detected at index " << index << "! Rehashing required." << std::endl;
           }
       }
       
       void display() {
           for (int i = 0; i < size; i++) {
               if (table[i] != -1) {
                   std::cout << "Index " << i << ": " << table[i] << std::endl;
               }
           }
       }
   };
   
   int main() {
       SimpleHash hashTable(7);
   
       hashTable.insert(10);
       hashTable.insert(20);
       hashTable.insert(5);
       hashTable.insert(33);
       hashTable.insert(42);
       hashTable.insert(17);
   
       hashTable.display();
   
       return 0;
   }
   
   ```

   Then back to the configuration python file `simple.py`:

   ```python
   system.mem_ctrl = MemCtrl()
   system.mem_ctrl.dram = DDR3_1600_8x8()
   system.mem_ctrl.dram.range = system.mem_ranges[0]
   system.mem_ctrl.port = system.membus.mem_side_ports
   ```

   We need to modify the memory controller to use DRAMSim3:

   ```python
   system.mem_ctrl = DRAMsim3()
   system.mem_ctrl.configFile = '/gem5/ext/dramsim3/DRAMsim3/configs/CXL_4Gb_x16_2400.ini'
   system.mem_ctrl.range = system.mem_ranges[0]
   system.mem_ctrl.port = system.membus.mem_side_ports
   ```

   I here renamed the config python file to `run_simplehash.py`, and run with following command:

   `/gem5/build/X86/gem5.opt /run_simplehash.py `

   We can get the output result in directory named `m5out`, including `config.ini`, `config.json` and `stats.txt`.
