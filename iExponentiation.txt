library ieee;
use ieee.std_logic_1164.all;
use ieee.numeric_std.all;


entity exponentiation is
	generic (
		C_block_size  : integer := 256
	);
	port (
		--input control
		valid_in	 : in STD_LOGIC; --monPro_start
		ready_in	 : out STD_LOGIC; --monPro_busy/ready

		--input data
		A 	       : in STD_LOGIC_VECTOR(C_block_size-1 downto 0);
		B 	       : in STD_LOGIC_VECTOR(C_block_size-1 downto 0);
        
		--ouput control
		ready_out	: in STD_LOGIC;
		valid_out	: out STD_LOGIC; --monPro_finished

		--output data
		result 		: out STD_LOGIC_VECTOR(C_block_size-1 downto 0);

		--modulus
		modulus 	: in STD_LOGIC_VECTOR(C_block_size-1 downto 0);

		--utility
		clk 		: in STD_LOGIC;
		reset_n 	: in STD_LOGIC
	);
end exponentiation;


architecture expBehave of exponentiation is
    type state is (IDLE, CALCULATING, FINISHED);
    signal current_state : state := IDLE ;
    signal next_state    : state := IDLE;
    signal sum                                  : std_logic_vector(C_block_size downto 0);
    signal sum_operand                          : std_logic_vector(C_block_size -1 downto 0);
    signal difference                           : std_logic_vector(C_block_size downto 0);
    signal counter, counter_next                : integer range 0 to C_block_size; -- størrelse på denne?
    signal P                                    : std_logic_vector(C_block_size downto 0);
    
begin
   -- monPro : entitiy work.monPro port map();

    process(clk, reset_n)
    begin
        if(reset_n = '0') then
            --reset stuff
            counter <= 0;
            next_state <= IDLE;
        else if(clk'event and clk = '1') then
            --clk stuff
                current_state <= next_state;
                counter        <= counter_next;
             end if;
        end if;
    end process;
    
    
    process(clk, A, B, counter, sum, difference)
    begin
        case current_state is
            when IDLE =>
                P <= (others => '0'); -- sjekk denne
                ready_in <= '1';
                if(valid_in = '1') then
                    counter <= 0;
                    next_state <= CALCULATING;
                end if;
            
            when CALCULATING =>
                counter_next <= counter + 1;
                if(A(counter) = '1') then 
                   --P <= std_logic_vector(unsigned(P) + unsigned(B));
                   sum_operand <= B;
                   P <= sum;
                end if;
                if (P(0) = '1') then 
					sum_operand <= modulus;
					P <= sum;
                end if;
                
                P <= '0' & P(C_block_size - 1 downto 0); -- bitshift
            
            when FINISHED =>
                if (P >= '0' & modulus) then 
                    result <= difference(C_block_size - 1 downto 0);
                else
                    result <= P(C_block_size - 1 downto 0);
                end if;
                next_state <= IDLE;
				
        end case;
    end process;
    sum 		<=	std_logic_vector(unsigned(P) + unsigned('0' & sum_operand)); --  + unsigned('0' & B));
    difference 	<= 	std_logic_vector(unsigned(P) - unsigned('0' & modulus));
end expBehave;