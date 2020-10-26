library ieee;
use ieee.std_logic_1164.all;
use ieee.numeric_std.all;

entity rsa_core is
	generic (
		-- Users to add parameters here
		C_BLOCK_SIZE          : integer := 256
	);
	port (
		-----------------------------------------------------------------------------
		-- Clocks and reset
		-----------------------------------------------------------------------------
		clk                    :  in std_logic;
		reset_n                :  in std_logic;




		-----------------------------------------------------------------------------
		-- Slave msgin interface
		-----------------------------------------------------------------------------
		-- Message that will be sent out is valid
		msgin_valid             : in std_logic;
		-- Slave ready to accept a new message
		msgin_ready             : out std_logic;
		-- Message that will be sent out of the rsa_msgin module
		msgin_data              :  in std_logic_vector(C_BLOCK_SIZE-1 downto 0);
		-- Indicates boundary of last packet
		msgin_last              :  in std_logic;

		-----------------------------------------------------------------------------
		-- Master msgout interface
		-----------------------------------------------------------------------------
		-- Message that will be sent out is valid
		msgout_valid            : out std_logic;
		-- Slave ready to accept a new message
		msgout_ready            :  in std_logic;
		-- Message that will be sent out of the rsa_msgin module
		msgout_data             : out std_logic_vector(C_BLOCK_SIZE-1 downto 0);
		-- Indicates boundary of last packet
		msgout_last             : out std_logic;

		-----------------------------------------------------------------------------
		-- Interface to the register block
		-----------------------------------------------------------------------------
		key_e_d                 :     in std_logic_vector(C_BLOCK_SIZE-1 downto 0);
		key_n                   :     in std_logic_vector(C_BLOCK_SIZE-1 downto 0);
	    key_rn                  :     in std_logic_vector(C_BLOCK_SIZE-1 downto 0);
		key_r2n                 :     in std_logic_vector(C_BLOCK_SIZE-1 downto 0);
		rsa_status              :     out std_logic_vector(31 downto 0)
	);
end rsa_core;


---HUSK Å LEGGE TIL key_rn OG key_r2n I rsa_core! og i testbenk-filene!!!
--Counter: Tilordninger må være counter_next <= counter -1. Nå er det counter = counter 
architecture rtl of rsa_core is
    
    type state is   (IDLE, PRE_CALC, CALC_MODEX, PP_CALC, AP_CALC, POST_CALC, FINISHED);
    signal curr_state, next_state              : state;
    signal counter, counter_next               : integer range 0 to C_block_size; -- størrelse på denne?
    signal monpro_started, monpro_started_next : std_logic;

     --Registers
    signal data_in_REG      : std_logic_vector(C_block_size - 1 downto 0);
    signal n_REG            : std_logic_vector(C_block_size - 1 downto 0);
    signal rn_REG           : std_logic_vector(C_block_size - 1 downto 0);
    signal r2n_REG          : std_logic_vector(C_block_size - 1 downto 0);
    signal key_REG          : std_logic_vector(C_block_size - 1 downto 0); 
    signal M_REG            : std_logic_vector(C_block_size - 1 downto 0);       
    signal Result_REG       : std_logic_vector(C_block_size - 1 downto 0);
    signal status_REG       : std_logic_vector(31 downto 0);
    
        --MONPRO grensesnitt
    signal MUX1_OUT                : std_logic_vector (C_block_size - 1 downto 0);
    signal MUX2_OUT                : std_logic_vector (C_block_size - 1 downto 0);
    signal MONPRO_start            : std_logic;
    signal MONPRO_finished         : std_logic;
    signal MONPRO_result           : std_logic_vector (C_block_size - 1 downto 0);

begin
    
  
    msgout_last  <= msgin_last; -- Denne er grei enn så lenge. Ser ikke helt hva annet vi skulle gjort med den.
	rsa_status   <= (others => '0'); --Status er bare null nå. Bør nok sees på.
    
    --Master reset signal and clocking
    --Må endre litt.
    process (clk, reset_n)
    begin
        if (reset_n = '0') then
            curr_state     <= IDLE;
            counter        <= 0;
            monpro_started <= '0';
        elsif (clk'event and clk = '1') then
            curr_state     <= next_state;           --Ta denne ut? Og bare kjøre tilordninger direkte?
            counter        <= counter_next;         --Denne er grei
            monpro_started <= monpro_started_next;
        end if;
    end process;
    
   process(clk, curr_state) -- endre denne. CLK, counter og litt sånt
   begin
        case curr_state is
            when IDLE =>
                if(msgin_valid = '1' and msgin_ready = '1') then
                   key_REG <= key_e_d; --load key
                   n_REG <= key_n;      --load modulus
                   r2n_REG <= key_r2n;
                   msgin_ready <= '0';  --Set as busy in
                   data_in_REG <= msgin_data; --load input data
                   rn_REG <= key_rn;
                   next_state <= PRE_CALC;
                   msgin_ready <= '0';      
                end if;
            
            --Transfers message into the Montgomery domain.
            when PRE_CALC =>
                MUX1_OUT <= data_in_REG;
                MUX2_OUT <= r2n_REG;
                MONPRO_start <= '1'; --sjekk at monpro deaktiverer denne
                if(MONPRO_finished = '1') then -- Bytte ut med while?
                    data_in_REG <= MONPRO_result; --Denne kan nok endres til bare data_in_REG, så sparer vi litt på den.
                    next_state <= CALC_MODEX;
                end if;
                 
           when CALC_MODEX =>
                --A = M_REG - M_REG holder på Mr mod n
                --e = key_REG
                --n = n_REG
                --
                M_REG <= rn_REG; -- sjekk hvordan dette er. Skal bare gjøres én gang ved oppstart.
                counter_next <= C_block_size - 1;
                next_state <= PP_CALC; --Overstyrer denne?
          
          
          --Disse må sees på. Ikke helt optimalt å ha to tilstander sånn.
          --Dette gjør at vi bruker et ekstra klokketick i hver overgang.
          when PP_CALC =>
                MUX1_OUT <= M_REG;
                MUX2_OUT <= M_REG;
                MONPRO_start <= '1';
                if(MONPRO_finished = '1') then
                    counter_next <= counter - 1;
                    MONPRO_start <= '0';
                    M_REG <= MONPRO_result;
                    if(key_REG(counter) = '1') then
                        next_state <= AP_CALC; -- Innafor å bare kjøre "curr_state <= AP_CALC"?
                    else if(counter = 0) then
                        next_state <= POST_CALC;
                        end if;
                    end if;
                end if;
                          
          when AP_CALC =>
                MUX1_OUT <= M_REG;
                MUX2_OUT <= data_in_REG;
                MONPRO_start <= '1';
                if(MONPRO_finished = '1') then
                    MONPRO_start <= '0';
                    M_REG <= MONPRO_result;
                    if(counter = 0) then
                        next_state <= POST_CALC;
                    else
                        next_state <= PP_CALC;
                    end if;
                end if;
     
           when POST_CALC =>     
                MUX1_OUT <= M_REG;
                MUX2_OUT <= (others => '1');
                MONPRO_start <= '1';
                msgout_valid <= '0'; -- Bare fordi jeg tror den må settes et sted. Kanskje ikke?
                if(MONPRO_finished = '1') then
                    MONPRO_start <= '0'; 
                    Result_REG <= MONPRO_result;
                    next_state <= FINISHED;
                end if;
                
            when FINISHED =>
                msgout_valid <= '1';
                if(msgout_ready = '1') then 
                    msgout_data <= Result_REG;
                end if;

        end case;
   end process; 

--Ikke endret fra tidligere Dette blir monPro, enn så lenge...
i_exponentiation : entity work.exponentiation
		generic map (
			C_block_size => C_BLOCK_SIZE
		)
		port map (
			A   => MUX1_OUT  ,
			B   => MUX2_OUT  ,
			valid_in  => msgin_valid ,
			ready_in  => MONPRO_start ,
			ready_out => MONPRO_finished,
			valid_out => msgout_valid,
			result    => MONPRO_result,
			modulus   => key_n       ,
			clk       => clk         ,
			reset_n   => reset_n
		);
end rtl;
    
--------------------------------------------------------------------------------
-- Author       : Oystein Gjermundnes
-- Organization : Norwegian University of Science and Technology (NTNU)
--                Department of Electronic Systems
--                https://www.ntnu.edu/ies
-- Course       : TFE4141 Design of digital systems 1 (DDS1)
-- Year         : 2018-2019
-- Project      : RSA accelerator
-- License      : This is free and unencumbered software released into the
--                public domain (UNLICENSE)
--------------------------------------------------------------------------------
-- Purpose:
--   RSA encryption core template. This core currently computes
--   C = M xor key_n
--
--   Replace/change this module so that it implements the function
--   C = M**key_e mod key_n.
--------------------------------------------------------------------------------
library ieee;
use ieee.std_logic_1164.all;
use ieee.numeric_std.all;

entity rsa_core is
	generic (
		-- Users to add parameters here
		C_BLOCK_SIZE          : integer := 256
	);
	port (
		-----------------------------------------------------------------------------
		-- Clocks and reset
		-----------------------------------------------------------------------------
		clk                    :  in std_logic;
		reset_n                :  in std_logic;




		-----------------------------------------------------------------------------
		-- Slave msgin interface
		-----------------------------------------------------------------------------
		-- Message that will be sent out is valid
		msgin_valid             : in std_logic;
		-- Slave ready to accept a new message
		msgin_ready             : out std_logic;
		-- Message that will be sent out of the rsa_msgin module
		msgin_data              :  in std_logic_vector(C_BLOCK_SIZE-1 downto 0);
		-- Indicates boundary of last packet
		msgin_last              :  in std_logic;

		-----------------------------------------------------------------------------
		-- Master msgout interface
		-----------------------------------------------------------------------------
		-- Message that will be sent out is valid
		msgout_valid            : out std_logic;
		-- Slave ready to accept a new message
		msgout_ready            :  in std_logic;
		-- Message that will be sent out of the rsa_msgin module
		msgout_data             : out std_logic_vector(C_BLOCK_SIZE-1 downto 0);
		-- Indicates boundary of last packet
		msgout_last             : out std_logic;

		-----------------------------------------------------------------------------
		-- Interface to the register block
		-----------------------------------------------------------------------------
		key_e_d                 :     in std_logic_vector(C_BLOCK_SIZE-1 downto 0);
		key_n                   :     in std_logic_vector(C_BLOCK_SIZE-1 downto 0);
	    key_rn                  :     in std_logic_vector(C_BLOCK_SIZE-1 downto 0);
		key_r2n                 :     in std_logic_vector(C_BLOCK_SIZE-1 downto 0);
		rsa_status              :     out std_logic_vector(31 downto 0)
	);
end rsa_core;


---HUSK Å LEGGE TIL key_rn OG key_r2n I rsa_core! og i testbenk-filene!!!
--Counter: Tilordninger må være counter_next <= counter -1. Nå er det counter = counter 
architecture rtl of rsa_core is
    
    type state is   (IDLE, PRE_CALC, CALC_MODEX, PP_CALC, AP_CALC, POST_CALC, FINISHED);
    signal curr_state, next_state              : state;
    signal counter, counter_next               : integer range 0 to C_block_size; -- størrelse på denne?
    signal monpro_started, monpro_started_next : std_logic;

     --Registers
    signal data_in_REG      : std_logic_vector(C_block_size - 1 downto 0);
    signal n_REG            : std_logic_vector(C_block_size - 1 downto 0);
    signal rn_REG           : std_logic_vector(C_block_size - 1 downto 0);
    signal r2n_REG          : std_logic_vector(C_block_size - 1 downto 0);
    signal key_REG          : std_logic_vector(C_block_size - 1 downto 0); 
    signal M_REG            : std_logic_vector(C_block_size - 1 downto 0);       
    signal Result_REG       : std_logic_vector(C_block_size - 1 downto 0);
    signal status_REG       : std_logic_vector(31 downto 0);
    
        --MONPRO grensesnitt
    signal MUX1_OUT                : std_logic_vector (C_block_size - 1 downto 0);
    signal MUX2_OUT                : std_logic_vector (C_block_size - 1 downto 0);
    signal MONPRO_start            : std_logic;
    signal MONPRO_finished         : std_logic;
    signal MONPRO_result           : std_logic_vector (C_block_size - 1 downto 0);

begin
    
  
    msgout_last  <= msgin_last; -- Denne er grei enn så lenge. Ser ikke helt hva annet vi skulle gjort med den.
	rsa_status   <= (others => '0'); --Status er bare null nå. Bør nok sees på.
    
    --Master reset signal and clocking
    --Må endre litt.
    process (clk, reset_n)
    begin
        if (reset_n = '0') then
            curr_state     <= IDLE;
            counter        <= 0;
            monpro_started <= '0';
        elsif (clk'event and clk = '1') then
            curr_state     <= next_state;           --Ta denne ut? Og bare kjøre tilordninger direkte?
            counter        <= counter_next;         --Denne er grei
            monpro_started <= monpro_started_next;
        end if;
    end process;
    
   process(clk, curr_state) -- endre denne. CLK, counter og litt sånt
   begin
        case curr_state is
            when IDLE =>
                if(msgin_valid = '1' and msgin_ready = '1') then
                   key_REG <= key_e_d; --load key
                   n_REG <= key_n;      --load modulus
                   r2n_REG <= key_r2n;
                   msgin_ready <= '0';  --Set as busy in
                   data_in_REG <= msgin_data; --load input data
                   rn_REG <= key_rn;
                   next_state <= PRE_CALC;
                   msgin_ready <= '0';        
                end if;
            
            --Transfers message into the Montgomery domain.
            when PRE_CALC =>
                MUX1_OUT <= data_in_REG;
                MUX2_OUT <= r2n_REG;
                MONPRO_start <= '1'; --sjekk at monpro deaktiverer denne
                if(MONPRO_finished = '1') then -- Bytte ut med while?
                    data_in_REG <= MONPRO_result; --Denne kan nok endres til bare data_in_REG, så sparer vi litt på den.
                    next_state <= CALC_MODEX;
                end if;
                 
           when CALC_MODEX =>
                --A = M_REG - M_REG holder på Mr mod n
                --e = key_REG
                --n = n_REG
                --
                M_REG <= rn_REG; -- sjekk hvordan dette er. Skal bare gjøres én gang ved oppstart.
                counter_next <= C_block_size - 1;
                next_state <= PP_CALC; --Overstyrer denne?
          
          
          --Disse må sees på. Ikke helt optimalt å ha to tilstander sånn.
          --Dette gjør at vi bruker et ekstra klokketick i hver overgang.
          when PP_CALC =>
                MUX1_OUT <= M_REG;
                MUX2_OUT <= M_REG;
                MONPRO_start <= '1';
                if(MONPRO_finished = '1') then
                    counter_next <= counter - 1;
                    MONPRO_start <= '0';
                    M_REG <= MONPRO_result;
                    if(key_REG(counter) = '1') then
                        next_state <= AP_CALC; -- Innafor å bare kjøre "curr_state <= AP_CALC"?
                    else if(counter = 0) then
                        next_state <= POST_CALC;
                        end if;
                    end if;
                end if;
                          
          when AP_CALC =>
                MUX1_OUT <= M_REG;
                MUX2_OUT <= data_in_REG;
                MONPRO_start <= '1';
                if(MONPRO_finished = '1') then
                    MONPRO_start <= '0';
                    M_REG <= MONPRO_result;
                    if(counter = 0) then
                        next_state <= POST_CALC;
                    else
                        next_state <= PP_CALC;
                    end if;
                end if;
     
           when POST_CALC =>     
                MUX1_OUT <= M_REG;
                MUX2_OUT <= (others => '1');
                MONPRO_start <= '1';
                msgout_valid <= '0'; -- Bare fordi jeg tror den må settes et sted. Kanskje ikke?
                if(MONPRO_finished = '1') then
                    MONPRO_start <= '0'; 
                    Result_REG <= MONPRO_result;
                    next_state <= FINISHED;
                end if;
                
            when FINISHED =>
                msgout_valid <= '1';
                if(msgout_ready = '1') then 
                    msgout_data <= Result_REG;
                end if;

        end case;
   end process; 

--Ikke endret fra tidligere Dette blir monPro, enn så lenge...
i_exponentiation : entity work.exponentiation
		generic map (
			C_block_size => C_BLOCK_SIZE
		)
		port map (
			A   => MUX1_OUT  ,
			B   => MUX2_OUT  ,
			valid_in  => msgin_valid ,
			ready_in  => MONPRO_start ,
			ready_out => MONPRO_finished,
			valid_out => msgout_valid,
			result    => MONPRO_result,
			modulus   => key_n       ,
			clk       => clk         ,
			reset_n   => reset_n
		);
end rtl;